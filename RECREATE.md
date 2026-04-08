# Debugging Agent: One-Shot Recreation Guide

> **What this is:** A complete, self-contained guide for an AI agent to recreate an automated debugging pipeline from scratch. Feed this entire file as a prompt to Claude Code (or any capable coding agent) and it will build the project for you.
>
> **What it does:** When a containerized service throws errors in production, this agent automatically fetches logs, clones the repo, uses Claude Code to diagnose and fix the bug, creates a draft PR, and posts the result to Slack.

---

## Architecture

```
Cloud Monitoring Alert (container service error)
    |
    +--> Slack channel  (human notification)
    |
    +--> Message Queue (push to your webhook)
              |
              v
         Container Service: "debugging-agent"
          |-- 1. Parse alert -> identify which service errored
          |-- 2. Fetch error logs from cloud logging API
          |-- 3. Clone the matching GitHub repo
          |-- 4. Run `claude -p` -> AI diagnoses bug, creates draft PR
          +-- 5. Post PR URL to Slack (or failure analysis)
```

---

## Cloud Platform Mapping

This project was originally built on GCP. Below is a mapping of every GCP service used to its equivalent on other platforms. **You must substitute the equivalent service for your platform throughout.**

| Role | GCP | AWS | Azure | Alibaba Cloud |
|---|---|---|---|---|
| **Container hosting** | Cloud Run | ECS Fargate / App Runner / Lambda (container) | Azure Container Apps / Azure Functions | Function Compute / SAE |
| **Monitoring/Alerts** | Cloud Monitoring | CloudWatch Alarms | Azure Monitor Alerts | CloudMonitor |
| **Log storage/query** | Cloud Logging | CloudWatch Logs | Azure Monitor Logs (Log Analytics) | SLS (Log Service) |
| **Message queue (push)** | Pub/Sub | SNS (with HTTP subscription) or EventBridge | Event Grid / Service Bus | MNS / EventBridge |
| **Secrets management** | Secret Manager | AWS Secrets Manager / SSM Parameter Store | Azure Key Vault | KMS |
| **Container registry** | Artifact Registry | ECR | ACR | ACR |
| **IAM / service identity** | Service Accounts + IAM | IAM Roles + Policies | Managed Identity + RBAC | RAM |

---

## Prerequisites

Before building, you need:

1. **Anthropic API key** - for Claude Code CLI (`sk-ant-...`)
2. **GitHub App** (preferred) or **GitHub PAT** - with `Contents` (read/write) + `Pull requests` (read/write) on the repos you want to monitor
3. **Slack Bot Token** - `xoxb-...` with `chat:write` scope, bot invited to the target channel
4. **Cloud CLI** installed and authenticated for your platform
5. **Claude Code CLI** - `npm install -g @anthropic-ai/claude-code`

### GitHub App Setup (Recommended over PAT)

A GitHub App gives you scoped, rotatable credentials without tying to a personal account.

1. Go to your GitHub org Settings > Developer settings > GitHub Apps > New GitHub App
2. Set:
   - Name: `debugging-agent` (or similar)
   - Homepage URL: anything
   - Uncheck "Webhook > Active" (not needed)
   - Permissions: Repository > Contents: Read & Write, Pull Requests: Read & Write
3. After creation, note the **App ID**
4. Generate a **Private Key** (.pem file) - save the contents
5. Install the App on your org, selecting only the repos you want to monitor
6. Note the **Installation ID** (visible in the URL when viewing the installation)

---

## File Structure

Create this exact structure:

```
debugging-agent/
|-- main.py              # Flask app - webhook endpoint + /health
|-- config.py            # Loads configs/config.yaml
|-- log_fetcher.py       # Cloud logging API - fetch error logs
|-- claude_runner.py     # Clone repo, run claude -p, extract PR URL
|-- slack_notifier.py    # Post success/failure to Slack
|-- configs/config.yaml  # GITIGNORED - secrets + service mapping
|-- requirements.txt     # Python dependencies
|-- Dockerfile           # Python 3.12 + Node.js 20 + claude CLI + gh CLI
|-- deploy.sh            # Infrastructure setup script (adapt to your cloud)
|-- .gitignore           # Must exclude configs/config.yaml
```

---

## File Contents

### `.gitignore`

```
configs/config.yaml
```

### `configs/config.yaml`

This is your secrets + configuration file. **Never commit this.** Store it in your cloud's secret manager and mount it into the container at runtime.

```yaml
# ---- Adapt this section to your cloud ----
# gcp_project is GCP-specific. For other clouds, replace with:
#   AWS: aws_region + account context (usually from env/IAM role)
#   Azure: subscription_id + resource_group
#   Alibaba: region_id + project
# The log_fetcher.py will use this to know where to query logs.
gcp_project: your-project-id

slack_channel: "#your-alerts-channel"

secrets:
  anthropic_api_key: "sk-ant-..."
  slack_bot_token: "xoxb-..."

# GitHub App credentials (recommended over PAT)
github_app:
  app_id: 1234567
  installation_id: 12345678
  private_key: |
    -----BEGIN RSA PRIVATE KEY-----
    ...your private key here...
    -----END RSA PRIVATE KEY-----

# Map of monitored services.
# Keys = the service name as it appears in alert payloads.
# Each value must have: repo, base_branch.
# Optional: code_owner (Slack member ID for @mentions),
#           and any cloud-specific fields like gcp_project.
services:
  my-api-service:
    repo: your-org/your-api-repo
    base_branch: main
    # code_owner: U12345678  # Slack member ID (optional)
    # gcp_project: other-project  # if different from default (GCP-specific)

  my-worker-service:
    repo: your-org/your-worker-repo
    base_branch: dev
    code_owner: U87654321
```

### `requirements.txt`

```
flask
gunicorn
slack_sdk
pyyaml
PyJWT[crypto]
requests
```

> **Cloud-specific dependency:** The original uses `google-cloud-logging` for GCP. Replace with your platform's SDK:
> - **AWS:** `boto3` (for CloudWatch Logs)
> - **Azure:** `azure-monitor-query`, `azure-identity`
> - **Alibaba:** `alibabacloud_sls20201230`
>
> Add the appropriate one to this file.

### `config.py`

```python
from pathlib import Path

import yaml

_CONFIG_PATH = Path(__file__).parent / "configs" / "config.yaml"

with open(_CONFIG_PATH) as f:
    _cfg = yaml.safe_load(f)

GCP_PROJECT: str = _cfg["gcp_project"]
SLACK_CHANNEL: str = _cfg["slack_channel"]
SERVICE_MAP: dict = _cfg["services"]

_secrets = _cfg["secrets"]
ANTHROPIC_API_KEY: str = _secrets["anthropic_api_key"]
SLACK_BOT_TOKEN: str = _secrets["slack_bot_token"]

_github_app = _cfg["github_app"]
GITHUB_APP_ID: int = _github_app["app_id"]
GITHUB_APP_INSTALLATION_ID: int = _github_app["installation_id"]
GITHUB_APP_PRIVATE_KEY: str = _github_app["private_key"]
```

> **Adapt:** Rename `GCP_PROJECT` to whatever your cloud identifier is (e.g., `AWS_REGION`, `AZURE_RESOURCE_GROUP`). If using a GitHub PAT instead of a GitHub App, replace the `github_app` block with a simple `github_token` string and simplify `claude_runner.py` accordingly.

### `main.py`

```python
import base64
import json
import logging
import time

from flask import Flask, request

from claude_runner import run_claude_fix
from config import SERVICE_MAP
from log_fetcher import fetch_error_logs
from slack_notifier import notify_failure, notify_success

logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s [%(levelname)s] %(name)s: %(message)s",
)
logger = logging.getLogger(__name__)

app = Flask(__name__)

# --- Duplicate-PR prevention ---
# In-memory cooldown: service_name -> last-processed unix timestamp.
# Resets on cold start; with max instances, worst-case is N PRs per window.
_cooldown: dict[str, float] = {}
COOLDOWN_SECONDS = 12 * 3600  # 12 hours

# Alert types that Claude can actually fix with code changes.
# Matched case-insensitively against the alert condition name.
_ACTIONABLE_KEYWORDS = ["error log", "http error", "5xx", "container crash"]


def _is_actionable_alert(condition_display_name: str) -> bool:
    """Return True if the alert type is something Claude can fix with code."""
    lower = condition_display_name.lower()
    return any(kw in lower for kw in _ACTIONABLE_KEYWORDS)


@app.route("/", methods=["POST"])
def handle_pubsub():
    """Receive a push message containing a cloud monitoring alert.

    ADAPT THIS for your cloud's message queue format:
    - GCP Pub/Sub: base64-encoded JSON in envelope.message.data
    - AWS SNS: JSON body with "Message" field (may be string-encoded JSON)
    - Azure Event Grid: JSON body with "data" field
    - Alibaba MNS: XML or JSON body depending on config
    """
    envelope = request.get_json(silent=True)
    if not envelope:
        logger.warning("Empty request body")
        return "Bad Request: no JSON", 400

    # ---- GCP Pub/Sub message decoding ----
    # Replace this block with your cloud's message format parsing
    pubsub_message = envelope.get("message", {})
    data_b64 = pubsub_message.get("data", "")

    try:
        alert_json = json.loads(base64.b64decode(data_b64))
    except Exception:
        logger.exception("Failed to decode message data")
        return "OK", 200  # Ack to avoid retries

    logger.info("Received alert: %s", json.dumps(alert_json, default=str)[:500])

    # ---- Alert payload parsing ----
    # GCP Cloud Monitoring uses this structure. Adapt for your platform:
    #   AWS CloudWatch: different JSON schema (detail.alarmName, etc.)
    #   Azure Monitor: different schema (data.essentials.monitorCondition, etc.)
    incident = alert_json.get("incident", {})
    service_name = (
        incident.get("resource", {}).get("labels", {}).get("service_name")
    )
    incident_summary = incident.get("summary", "No summary provided")
    incident_id = incident.get("incident_id", "unknown")

    # --- Guard 1: skip closed/resolved incidents ---
    if incident.get("state") == "closed":
        logger.info("Incident %s is closed - skipping", incident_id)
        return "OK", 200

    if not service_name:
        logger.warning("No service_name in alert payload - skipping")
        return "OK", 200

    if service_name not in SERVICE_MAP:
        logger.warning("Service %s not in SERVICE_MAP - skipping", service_name)
        return "OK", 200

    # --- Guard 2: only process actionable alert types ---
    condition_name = incident.get("condition", {}).get("displayName", "")
    if condition_name and not _is_actionable_alert(condition_name):
        logger.info(
            "Alert type '%s' for %s is not actionable - skipping",
            condition_name,
            service_name,
        )
        return "OK", 200

    # --- Guard 3: per-service cooldown (12 h) ---
    now = time.time()
    last = _cooldown.get(service_name, 0)
    if now - last < COOLDOWN_SECONDS:
        elapsed_min = int((now - last) / 60)
        logger.info(
            "Service %s on cooldown (last processed %d min ago) - skipping",
            service_name,
            elapsed_min,
        )
        return "OK", 200

    service_config = SERVICE_MAP[service_name]
    code_owner = service_config.get("code_owner")

    # Record cooldown *before* processing so concurrent requests are blocked too
    _cooldown[service_name] = now

    try:
        _process_alert(
            service_name=service_name,
            repo=service_config["repo"],
            base_branch=service_config["base_branch"],
            gcp_project=service_config.get("gcp_project"),
            incident_id=incident_id,
            incident_summary=incident_summary,
            code_owner=code_owner,
        )
    except Exception:
        logger.exception("Unhandled error processing alert for %s", service_name)
        notify_failure(
            service_name=service_name,
            incident_summary=incident_summary,
            analysis="Internal error in debugging-agent. Check container logs.",
            code_owner=code_owner,
        )

    # Always return 200 to prevent message queue retries (avoids duplicate PRs)
    return "OK", 200


def _process_alert(
    service_name: str,
    repo: str,
    base_branch: str,
    incident_id: str,
    incident_summary: str,
    gcp_project: str | None = None,
    code_owner: str | None = None,
) -> None:
    """Orchestrate the full fix pipeline: logs -> claude -> slack."""
    # 1. Fetch error logs
    logger.info("Fetching error logs for %s", service_name)
    error_logs = fetch_error_logs(service_name, gcp_project=gcp_project)

    # 2. Run Claude to fix the bug
    logger.info("Running Claude fix for %s (incident=%s)", service_name, incident_id)
    result = run_claude_fix(
        repo=repo,
        base_branch=base_branch,
        service_name=service_name,
        incident_id=incident_id,
        incident_summary=incident_summary,
        error_logs=error_logs,
    )

    # 3. Post result to Slack
    if result["success"] and result["pr_url"]:
        notify_success(
            service_name=service_name,
            incident_summary=incident_summary,
            pr_url=result["pr_url"],
            code_owner=code_owner,
        )
    else:
        notify_failure(
            service_name=service_name,
            incident_summary=incident_summary,
            analysis=result.get("analysis", "No analysis available."),
            code_owner=code_owner,
        )


@app.route("/health", methods=["GET"])
def health():
    return "OK", 200


if __name__ == "__main__":
    app.run(host="0.0.0.0", port=8080, debug=True)
```

### `log_fetcher.py`

This is **the most cloud-specific file**. Below is the GCP version, followed by adaptation guides for each cloud.

#### GCP Version (original)

```python
import logging
from datetime import datetime, timedelta, timezone

from google.cloud import logging as cloud_logging

from config import GCP_PROJECT

logger = logging.getLogger(__name__)


def fetch_error_logs(
    service_name: str,
    lookback_minutes: int = 10,
    gcp_project: str | None = None,
) -> str:
    """Fetch recent error logs for a Cloud Run service from Cloud Logging."""
    project = gcp_project or GCP_PROJECT
    client = cloud_logging.Client(project=project)

    timestamp_cutoff = (
        datetime.now(timezone.utc) - timedelta(minutes=lookback_minutes)
    ).strftime("%Y-%m-%dT%H:%M:%SZ")

    filter_str = (
        f'resource.type="cloud_run_revision" '
        f'resource.labels.service_name="{service_name}" '
        f'resource.labels.project_id="{project}" '
        f"severity>=ERROR "
        f'timestamp>="{timestamp_cutoff}"'
    )

    logger.info("Fetching logs with filter: %s", filter_str)

    entries = []
    for entry in client.list_entries(
        filter_=filter_str,
        order_by=cloud_logging.DESCENDING,
        max_results=50,
    ):
        timestamp = entry.timestamp.isoformat() if entry.timestamp else "unknown"
        severity = entry.severity or "ERROR"
        payload = entry.payload if isinstance(entry.payload, str) else str(entry.payload)
        entries.append(f"[{timestamp}] [{severity}] {payload}")

    if not entries:
        return "No error logs found in the specified time window."

    return "\n".join(entries)
```

#### AWS CloudWatch Logs Version

```python
import logging
from datetime import datetime, timedelta, timezone

import boto3

logger = logging.getLogger(__name__)


def fetch_error_logs(
    service_name: str,
    lookback_minutes: int = 10,
    **kwargs,
) -> str:
    """Fetch recent error logs for an ECS/App Runner service from CloudWatch."""
    client = boto3.client("logs")

    # Adjust log group name pattern to match your setup:
    # ECS: /ecs/{service_name}
    # App Runner: /aws/apprunner/{service_name}/application
    # Lambda: /aws/lambda/{service_name}
    log_group = f"/ecs/{service_name}"

    start_time = int(
        (datetime.now(timezone.utc) - timedelta(minutes=lookback_minutes)).timestamp() * 1000
    )
    end_time = int(datetime.now(timezone.utc).timestamp() * 1000)

    try:
        response = client.filter_log_events(
            logGroupName=log_group,
            startTime=start_time,
            endTime=end_time,
            filterPattern="ERROR",
            limit=50,
        )
    except client.exceptions.ResourceNotFoundException:
        return f"Log group {log_group} not found."

    entries = []
    for event in response.get("events", []):
        ts = datetime.fromtimestamp(event["timestamp"] / 1000, tz=timezone.utc).isoformat()
        entries.append(f"[{ts}] [ERROR] {event['message'].strip()}")

    if not entries:
        return "No error logs found in the specified time window."

    return "\n".join(entries)
```

#### Azure Monitor Logs Version

```python
import logging
from datetime import datetime, timedelta, timezone

from azure.identity import DefaultAzureCredential
from azure.monitor.query import LogsQueryClient, LogsQueryStatus

logger = logging.getLogger(__name__)


def fetch_error_logs(
    service_name: str,
    lookback_minutes: int = 10,
    **kwargs,
) -> str:
    """Fetch recent error logs from Azure Monitor (Log Analytics)."""
    credential = DefaultAzureCredential()
    client = LogsQueryClient(credential)

    # Set your Log Analytics workspace ID in config.yaml
    # or pass via kwargs / environment variable
    workspace_id = kwargs.get("workspace_id", "YOUR_WORKSPACE_ID")

    query = f"""
    ContainerAppConsoleLogs_CL
    | where ContainerAppName_s == "{service_name}"
    | where Log_s contains "ERROR" or Log_s contains "Exception"
    | order by TimeGenerated desc
    | take 50
    | project TimeGenerated, Log_s
    """

    end_time = datetime.now(timezone.utc)
    start_time = end_time - timedelta(minutes=lookback_minutes)

    response = client.query_workspace(
        workspace_id=workspace_id,
        query=query,
        timespan=(start_time, end_time),
    )

    if response.status != LogsQueryStatus.SUCCESS:
        return f"Log query failed: {response.partial_error}"

    entries = []
    for row in response.tables[0].rows:
        entries.append(f"[{row[0].isoformat()}] [ERROR] {row[1]}")

    if not entries:
        return "No error logs found in the specified time window."

    return "\n".join(entries)
```

#### Alibaba Cloud SLS Version

```python
import logging
from datetime import datetime, timedelta, timezone

from alibabacloud_sls20201230.client import Client
from alibabacloud_tea_openapi.models import Config

logger = logging.getLogger(__name__)


def fetch_error_logs(
    service_name: str,
    lookback_minutes: int = 10,
    **kwargs,
) -> str:
    """Fetch recent error logs from Alibaba Cloud SLS."""
    config = Config(
        access_key_id="from-config-or-env",
        access_key_secret="from-config-or-env",
        endpoint="your-region.log.aliyuncs.com",
    )
    client = Client(config)

    end_time = int(datetime.now(timezone.utc).timestamp())
    start_time = end_time - (lookback_minutes * 60)

    # Adapt project/logstore names to your setup
    query = f'* and service:{service_name} and level:ERROR | select __time__, content order by __time__ desc limit 50'

    response = client.get_logs(
        project="your-sls-project",
        logstore="your-logstore",
        _from=start_time,
        to=end_time,
        query=query,
    )

    entries = []
    for log in response.body:
        ts = datetime.fromtimestamp(int(log.get("__time__", 0)), tz=timezone.utc).isoformat()
        entries.append(f"[{ts}] [ERROR] {log.get('content', '')}")

    if not entries:
        return "No error logs found in the specified time window."

    return "\n".join(entries)
```

### `claude_runner.py`

This file is **cloud-agnostic** -- it only talks to GitHub and the local Claude CLI.

```python
import json
import logging
import os
import re
import shutil
import subprocess
import tempfile
import time

import jwt
import requests

from config import (
    ANTHROPIC_API_KEY,
    GITHUB_APP_ID,
    GITHUB_APP_INSTALLATION_ID,
    GITHUB_APP_PRIVATE_KEY,
)

logger = logging.getLogger(__name__)


def _get_installation_token() -> str:
    """Generate a short-lived GitHub installation access token from the App credentials."""
    now = int(time.time())
    payload = {"iat": now - 60, "exp": now + 600, "iss": str(GITHUB_APP_ID)}
    jwt_token = jwt.encode(payload, GITHUB_APP_PRIVATE_KEY, algorithm="RS256")

    resp = requests.post(
        f"https://api.github.com/app/installations/{GITHUB_APP_INSTALLATION_ID}/access_tokens",
        headers={
            "Authorization": f"Bearer {jwt_token}",
            "Accept": "application/vnd.github+json",
        },
        timeout=15,
    )
    resp.raise_for_status()
    return resp.json()["token"]


def run_claude_fix(
    repo: str,
    base_branch: str,
    service_name: str,
    incident_id: str,
    incident_summary: str,
    error_logs: str,
) -> dict:
    """Clone a repo, run claude -p to fix the bug, and return the result.

    Returns a dict with:
      - success: bool
      - pr_url: str or None
      - analysis: str (Claude's analysis / output summary)
    """
    github_token = _get_installation_token()
    branch_name = f"fix/{incident_id}"

    # Deduplication: skip if a branch already exists for this incident
    if _branch_exists(repo, branch_name, github_token):
        logger.info("Branch %s already exists for %s - skipping duplicate", branch_name, repo)
        return {
            "success": False,
            "pr_url": None,
            "analysis": f"Skipped: branch {branch_name} already exists (duplicate alert).",
        }

    workdir = tempfile.mkdtemp(prefix="debug-agent-")

    try:
        # Clone the repo
        clone_url = f"https://x-access-token:{github_token}@github.com/{repo}.git"
        logger.info("Cloning %s into %s", repo, workdir)
        subprocess.run(
            ["git", "clone", "--depth", "50", "-b", base_branch, clone_url, workdir],
            check=True,
            capture_output=True,
            text=True,
            timeout=120,
        )

        # Configure git identity for commits
        subprocess.run(
            ["git", "config", "user.email", "debugging-agent[bot]@users.noreply.github.com"],
            cwd=workdir, check=True, capture_output=True,
        )
        subprocess.run(
            ["git", "config", "user.name", "debugging-agent[bot]"],
            cwd=workdir, check=True, capture_output=True,
        )

        # Build the prompt for Claude
        prompt = _build_prompt(
            service_name, incident_summary, error_logs, branch_name, base_branch,
        )

        # Run claude -p
        logger.info("Running claude -p for service=%s incident=%s", service_name, incident_id)
        env = {
            **os.environ,
            "ANTHROPIC_API_KEY": ANTHROPIC_API_KEY,
            "GITHUB_TOKEN": github_token,
        }

        result = subprocess.run(
            [
                "claude", "-p", prompt,
                "--allowedTools", "Bash,Read,Edit,Write,Glob,Grep",
                "--max-turns", "30",
                "--output-format", "json",
                "--append-system-prompt",
                "Always create draft PRs. Use gh CLI for PR creation. "
                "Authenticate gh using the GITHUB_TOKEN env var: "
                "run `echo $GITHUB_TOKEN | gh auth login --with-token` first.",
            ],
            cwd=workdir,
            env=env,
            capture_output=True,
            text=True,
            timeout=780,  # 13 min - leave buffer under the 15 min container timeout
        )

        stdout = result.stdout
        stderr = result.stderr

        if result.returncode != 0:
            logger.error("claude -p failed: %s", stderr)
            return {
                "success": False,
                "pr_url": None,
                "analysis": f"Claude CLI exited with code {result.returncode}.\n{stderr[:2000]}",
            }

        # Parse output - try JSON first, fall back to text
        analysis = _extract_analysis(stdout)

        # Check if Claude determined no fix was needed
        if "NO_FIX_NEEDED" in analysis:
            logger.info("Claude determined no fix needed: %s", analysis[:200])
            return {"success": False, "pr_url": None, "analysis": analysis}

        pr_url = _extract_pr_url(stdout)

        if pr_url:
            logger.info("PR created: %s", pr_url)
            return {"success": True, "pr_url": pr_url, "analysis": analysis}

        logger.warning("No PR URL found in Claude output")
        return {
            "success": False,
            "pr_url": None,
            "analysis": analysis or "Claude ran but no PR was created.",
        }

    except subprocess.TimeoutExpired:
        logger.error("Timed out running claude -p")
        return {
            "success": False,
            "pr_url": None,
            "analysis": "Claude session timed out after 13 minutes.",
        }
    except Exception as e:
        logger.exception("Unexpected error in claude_runner")
        return {
            "success": False,
            "pr_url": None,
            "analysis": f"Internal error: {e}",
        }
    finally:
        shutil.rmtree(workdir, ignore_errors=True)


def _build_prompt(
    service_name: str,
    incident_summary: str,
    error_logs: str,
    branch_name: str,
    base_branch: str,
) -> str:
    return f"""You are debugging a production container service.

**Service:** {service_name}
**Alert Summary:** {incident_summary}
**Error Logs:**
```
{error_logs}
```

CRITICAL RULES:
- ONLY fix the specific error described in the alert summary and error logs above.
- Do NOT look for other issues, do NOT refactor code, do NOT make improvements unrelated to this error.
- If the error logs are empty or say "No error logs found", respond with "NO_FIX_NEEDED: No error logs available to diagnose." and STOP. Do not proceed.
- If after reading the codebase you cannot identify the root cause from the logs, respond with "NO_FIX_NEEDED: Could not identify root cause from the provided logs." and STOP.

Instructions (only if you can identify the specific error):
1. Read the codebase to understand the service architecture.
2. Identify the root cause of the specific error shown in the logs.
3. Fix ONLY that bug - nothing else.
4. Create a new branch named `{branch_name}`.
5. Commit your changes with a clear commit message.
6. Push the branch to origin.
7. Create a **draft** PR targeting the `{base_branch}` branch with:
   - A title describing the fix
   - A body explaining: what was broken, root cause, and what you fixed
8. Output the PR URL at the end of your response.
"""


def _branch_exists(repo: str, branch_name: str, token: str) -> bool:
    """Check if a branch already exists on the remote repo."""
    try:
        result = subprocess.run(
            ["git", "ls-remote", "--heads",
             f"https://x-access-token:{token}@github.com/{repo}.git",
             branch_name],
            capture_output=True, text=True, timeout=30,
        )
        return bool(result.stdout.strip())
    except Exception:
        return False


def _extract_pr_url(output: str) -> str | None:
    """Extract a GitHub PR URL from Claude's output."""
    try:
        data = json.loads(output)
        text = data.get("result", "") if isinstance(data, dict) else str(data)
    except (json.JSONDecodeError, TypeError):
        text = output

    match = re.search(r"https://github\.com/[^\s\"']+/pull/\d+", text)
    return match.group(0) if match else None


def _extract_analysis(output: str) -> str:
    """Extract a human-readable summary from Claude's output."""
    try:
        data = json.loads(output)
        if isinstance(data, dict):
            return data.get("result", output[:3000])
    except (json.JSONDecodeError, TypeError):
        pass
    return output[-3000:] if len(output) > 3000 else output
```

> **If using a GitHub PAT instead of a GitHub App:** Replace `_get_installation_token()` with a function that simply returns the PAT from config. Remove the `jwt` and `requests` imports, and remove `PyJWT[crypto]` and `requests` from requirements.txt if not used elsewhere.

### `slack_notifier.py`

This file is **cloud-agnostic** -- it only talks to the Slack API.

```python
import logging

from slack_sdk import WebClient
from slack_sdk.errors import SlackApiError

from config import SLACK_BOT_TOKEN, SLACK_CHANNEL

logger = logging.getLogger(__name__)


def _get_client() -> WebClient:
    return WebClient(token=SLACK_BOT_TOKEN)


def notify_success(
    service_name: str,
    incident_summary: str,
    pr_url: str,
    code_owner: str | None = None,
) -> None:
    """Post a success message to Slack with the PR link."""
    owner_line = f"\n*Owner:* <@{code_owner}> - please review" if code_owner else ""
    text = (
        f":wrench: *Auto-fix PR created*\n"
        f"*Service:* `{service_name}`\n"
        f"*Alert:* {incident_summary}\n"
        f"*PR:* {pr_url}\n"
        f"Status: Draft - ready for review"
        f"{owner_line}"
    )
    _post_message(text)


def notify_failure(
    service_name: str,
    incident_summary: str,
    analysis: str,
    code_owner: str | None = None,
) -> None:
    """Post a failure message to Slack so a human can follow up."""
    if len(analysis) > 1500:
        analysis = analysis[:1500] + "\n... (truncated)"

    owner_line = f"\n*Owner:* <@{code_owner}> - needs your attention" if code_owner else ""
    text = (
        f":warning: *Auto-fix failed - needs human review*\n"
        f"*Service:* `{service_name}`\n"
        f"*Alert:* {incident_summary}\n"
        f"*Analysis:*\n```\n{analysis}\n```"
        f"{owner_line}"
    )
    _post_message(text)


def _post_message(text: str) -> None:
    """Send a message to the configured Slack channel."""
    try:
        client = _get_client()
        client.chat_postMessage(channel=SLACK_CHANNEL, text=text)
        logger.info("Slack message sent to %s", SLACK_CHANNEL)
    except SlackApiError as e:
        logger.error("Failed to send Slack message: %s", e.response["error"])
```

### `Dockerfile`

```dockerfile
FROM python:3.12-slim

# Install git, Node.js (for claude CLI), gh CLI
RUN apt-get update && apt-get install -y --no-install-recommends \
        git curl gnupg ca-certificates \
    && curl -fsSL https://deb.nodesource.com/setup_20.x | bash - \
    && apt-get install -y --no-install-recommends nodejs \
    && npm install -g @anthropic-ai/claude-code \
    && curl -fsSL https://cli.github.com/packages/githubcli-archive-keyring.gpg \
       | dd of=/usr/share/keyrings/githubcli-archive-keyring.gpg \
    && echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/githubcli-archive-keyring.gpg] https://cli.github.com/packages stable main" \
       | tee /etc/apt/sources.list.d/github-cli.list > /dev/null \
    && apt-get update && apt-get install -y --no-install-recommends gh \
    && apt-get clean && rm -rf /var/lib/apt/lists/*

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

CMD ["gunicorn", "--bind", "0.0.0.0:8080", "--timeout", "900", "--workers", "1", "main:app"]
```

### `deploy.sh`

This is the **most cloud-specific file**. Below is the GCP version, followed by guidance for other clouds.

#### GCP Version (original)

```bash
#!/usr/bin/env bash
set -euo pipefail

PROJECT="your-gcp-project"
REGION="your-region"
SERVICE="debugging-agent"
SA="${SERVICE}-sa"
TOPIC="cloud-run-alerts"
CONFIG_SECRET="debugging-agent-config"

echo "=== 1. Create service account ==="
gcloud iam service-accounts create "$SA" \
  --display-name="Debugging Agent SA" \
  --project="$PROJECT" 2>/dev/null || echo "Already exists"

SA_EMAIL="${SA}@${PROJECT}.iam.gserviceaccount.com"

echo "=== 2. Grant IAM roles ==="
for ROLE in roles/logging.viewer roles/run.invoker roles/secretmanager.secretAccessor; do
  gcloud projects add-iam-policy-binding "$PROJECT" \
    --member="serviceAccount:${SA_EMAIL}" --role="$ROLE" --quiet
done

echo "=== 3. Create Pub/Sub topic ==="
gcloud pubsub topics create "$TOPIC" --project="$PROJECT" 2>/dev/null || echo "Already exists"

echo "=== 4. Create push subscription ==="
CLOUD_RUN_URL=$(gcloud run services describe "$SERVICE" \
  --region="$REGION" --project="$PROJECT" --format='value(status.url)')

gcloud pubsub subscriptions create "${SERVICE}-sub" \
  --topic="$TOPIC" --push-endpoint="$CLOUD_RUN_URL" \
  --push-auth-service-account="$SA_EMAIL" --ack-deadline=600 \
  --project="$PROJECT" 2>/dev/null || echo "Already exists"

echo ""
echo "NEXT STEPS:"
echo "  1. Upload configs/config.yaml to Secret Manager"
echo "  2. Deploy Cloud Run with secret mounted at /app/configs/config.yaml"
echo "  3. Add '$TOPIC' as notification channel on alert policies"
```

#### AWS Equivalent Setup

```bash
#!/usr/bin/env bash
set -euo pipefail

REGION="us-east-1"
SERVICE="debugging-agent"
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

# 1. Create ECR repo and push image
aws ecr create-repository --repository-name $SERVICE --region $REGION 2>/dev/null || true
# docker build, tag, push ...

# 2. Store secrets in SSM or Secrets Manager
aws secretsmanager create-secret --name debugging-agent-config \
  --secret-string file://configs/config.yaml --region $REGION

# 3. Create SNS topic
TOPIC_ARN=$(aws sns create-topic --name cloud-run-alerts --query TopicArn --output text)

# 4. Deploy via App Runner / ECS / Lambda and get the endpoint URL
# Then subscribe the endpoint to the SNS topic:
aws sns subscribe --topic-arn $TOPIC_ARN --protocol https \
  --notification-endpoint "https://your-service-url.amazonaws.com"

# 5. Create CloudWatch Alarms that publish to the SNS topic
```

#### Azure Equivalent Setup

```bash
#!/usr/bin/env bash
set -euo pipefail

RG="your-resource-group"
LOCATION="eastus"
SERVICE="debugging-agent"

# 1. Create Container App Environment + App
az containerapp env create --name debug-env --resource-group $RG --location $LOCATION
# az containerapp create ...

# 2. Store secrets in Key Vault
az keyvault secret set --vault-name your-vault --name debugging-agent-config \
  --file configs/config.yaml

# 3. Create Event Grid topic
az eventgrid topic create --name cloud-run-alerts --resource-group $RG --location $LOCATION

# 4. Subscribe your container endpoint to Event Grid
az eventgrid event-subscription create --name debug-sub \
  --source-resource-id "/subscriptions/.../topics/cloud-run-alerts" \
  --endpoint "https://your-container-app-url/api/events"

# 5. Create Azure Monitor alert rules targeting the Event Grid topic
```

---

## Infrastructure Requirements Checklist

Regardless of cloud platform, you need these components:

### 1. Container Service
- **Image:** Python 3.12 + Node.js 20 + `@anthropic-ai/claude-code` + `gh` CLI + `git`
- **Timeout:** 900 seconds (15 min) -- Claude needs time to analyze and fix
- **Memory:** 2 GiB minimum
- **CPU:** 1 vCPU
- **Max instances:** 3 (each request is long-running)
- **Workers:** 1 gunicorn worker (not more -- each request takes minutes)
- **Port:** 8080
- **Health check:** `GET /health`
- **Authentication:** Require auth (only the message queue should invoke it)

### 2. Secrets
- Mount `configs/config.yaml` into the container at `/app/configs/config.yaml`
- The container reads this file at startup

### 3. Message Queue
- Create a topic/queue for alerts
- Create a **push subscription** that POSTs to the container's URL
- Set ack deadline / visibility timeout to **600 seconds** (10 min)
- The subscription must authenticate (service account / IAM role)

### 4. Monitoring Alerts
For each monitored service, create alerts for these conditions. Only "Error Logs", "HTTP Errors", and "Container Crashes" are actionable by the agent (it will auto-skip the rest), but all are useful for Slack visibility.

| Alert | What to monitor | Threshold |
|---|---|---|
| **Error Logs** | Log entries with severity >= ERROR | > 3 in 60s |
| **HTTP Errors** | Request count with 5xx status | > 0 in 60s |
| **Container Crashes** | Log entries with CRITICAL/EMERGENCY severity | > 0 in 60s |
| **High Memory** | Memory utilization | > 80% sustained 2 min |
| **High CPU** | CPU utilization | > 80% sustained 2 min |
| **High Latency** | Request latency P95 | > 30s sustained 2 min |
| **Max Instances** | Active instance count | > 80% of max sustained 5 min |

Each alert must notify both:
- Your Slack channel (for humans)
- The message queue topic (for the debugging agent)

### 5. IAM / Permissions
The container's runtime identity needs:
- **Read logs** from all monitored services (including cross-project/cross-account if applicable)
- **Read secrets** from your secrets manager
- **Network egress** to: GitHub API, Slack API, Anthropic API

The message queue's push identity needs:
- **Invoke** the container service

### 6. Cross-Account/Cross-Project Monitoring
If you monitor services in multiple cloud accounts/projects:
- Grant the agent's runtime identity log-read access in each remote account
- Grant each remote account's monitoring/alerting service permission to publish to the agent's message queue topic
- Create alert policies in each remote account pointing to the agent's message queue

---

## Guard Rails

These are important safety mechanisms built into the system:

| Guard Rail | What it does | Why |
|---|---|---|
| **Draft PRs only** | Claude creates draft PRs, never merges | Human must review before merge |
| **12-hour cooldown** | Same service won't be processed twice in 12h | Prevents duplicate PRs from repeated alerts |
| **Actionable filter** | Only error/crash alerts trigger Claude | High CPU/memory/latency can't be fixed with code |
| **Branch dedup** | Skips if fix branch already exists | Prevents duplicate work from parallel alerts |
| **Max 30 turns** | Claude CLI limited to 30 tool calls | Prevents runaway sessions |
| **13-min timeout** | Subprocess timeout on Claude CLI | Leaves buffer under 15-min container timeout |
| **Always ack (200)** | Always returns 200 to message queue | Prevents retry storms creating duplicate PRs |
| **NO_FIX_NEEDED** | Claude stops if logs are empty or unclear | Prevents speculative/wrong fixes |

---

## Testing

### Publish a test alert to your message queue

#### GCP
```bash
gcloud pubsub topics publish cloud-run-alerts --project=your-project --message='{
  "incident": {
    "incident_id": "test-001",
    "summary": "Test alert - ignore",
    "state": "open",
    "resource": {
      "labels": {
        "service_name": "your-service-name"
      }
    },
    "condition": {
      "displayName": "Error Logs"
    }
  }
}'
```

#### AWS (SNS)
```bash
aws sns publish --topic-arn arn:aws:sns:REGION:ACCOUNT:cloud-run-alerts --message '{
  "incident": {
    "incident_id": "test-001",
    "summary": "Test alert - ignore",
    "state": "open",
    "resource": { "labels": { "service_name": "your-service-name" } },
    "condition": { "displayName": "Error Logs" }
  }
}'
```

After publishing, check:
1. Container logs for processing output
2. Your Slack channel for the success/failure message

---

## Quick Start Summary

1. **Create the files** listed above (adapt `log_fetcher.py` and `deploy.sh` for your cloud)
2. **Fill in `configs/config.yaml`** with your secrets and service mapping
3. **Upload the secret** to your cloud's secret manager
4. **Build and deploy** the Docker image to your container service, mounting the secret
5. **Run `deploy.sh`** (adapted) to create the message queue and IAM bindings
6. **Create alert policies** for each monitored service, notifying both Slack and the message queue
7. **Test** with a fake alert message
8. **Invite** the Slack bot to your alerts channel

That's it. When a monitored service errors, the pipeline fires automatically.
