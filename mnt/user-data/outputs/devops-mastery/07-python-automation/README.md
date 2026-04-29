# Module 7 — Python automation & backend services

> *"Building... backend automation scripts using Python."*  
> *"Developed and maintained scalable backend services... for cloud-native applications serving a large user base."*

This module is about the Python you write *as a DevOps engineer*. That's a different skill set from the Python a backend dev writes. Yours is mostly:

- Glue between APIs, CLIs, and CI systems
- Automation that needs to be reliable but can be slow
- Tooling that runs in CI/CD or on a schedule
- Small backend services that *support* your infrastructure (e.g. webhook handlers, metric exporters, cleanup workers)

It rewards a different aesthetic: short, robust, observable, easy to debug at 2am. Not particularly fast.

## The DevOps Python stack (a curated list)

These are libraries and tools you should know by name. You don't need to know all of them deeply, but you should know which one to reach for.

### HTTP and APIs
- `httpx` — modern HTTP client, supports sync and async. Prefer over `requests` for new code.
- `tenacity` — retry decorators with backoff. Essential for talking to flaky APIs.
- `pydantic` — data validation. Makes API responses safe to work with.

### CLI tools
- `typer` — modern CLI framework, type-driven. Or `click` for the older style.
- `rich` — pretty terminal output, progress bars, tables, syntax highlighting.

### Web services
- `fastapi` — async-first web framework. Fast, type-driven, auto-generates OpenAPI docs.
- `uvicorn` — ASGI server.
- `gunicorn` — process manager, often with uvicorn workers in production.

### Cloud SDKs
- `boto3` — AWS. The official AWS SDK.
- `google-cloud-*` — GCP service-specific clients.
- `kubernetes` — official K8s Python client.
- `pygithub` or `httpx` directly — GitHub API.

### Quality tools
- `ruff` — linter and formatter, written in Rust, very fast. Replaces flake8/black/isort.
- `mypy` or `pyright` — type checker.
- `pytest` — test framework.
- `pip-tools` or `uv` — dependency management.

### Packaging and distribution
- `uv` — modern, fast all-in-one Python package manager. Strong choice for new projects.
- `poetry` — older but more featured. Good if you need wheel building.
- Plain `pip` + `requirements.txt` — still fine for small scripts.

## Tenets of good DevOps Python

### 1. Make it idempotent

A script that breaks if you run it twice will eventually break. Always check current state before changing it.

```python
# Bad
def add_user_to_team(team, user):
    api.add_member(team, user)  # fails if user already in team

# Good
def add_user_to_team(team, user):
    members = api.get_members(team)
    if user not in members:
        api.add_member(team, user)
```

### 2. Fail loudly, log helpfully

Suppressing exceptions to "make the script pass" creates monsters that look successful but corrupt data.

```python
# Bad
try:
    do_thing()
except Exception:
    pass

# Good
try:
    do_thing()
except SpecificExpectedError as e:
    logger.warning("expected condition: %s, continuing", e)
except Exception:
    logger.exception("unexpected error in do_thing")
    raise
```

Always use `logger.exception` (not `logger.error`) when re-raising — it includes the traceback.

### 3. Use type hints, run mypy

Type hints catch entire classes of bugs at lint time and double as documentation.

```python
def fetch_pull_requests(
    repo: str,
    state: Literal["open", "closed", "all"] = "open",
    limit: int | None = None,
) -> list[PullRequest]:
    ...
```

Reading this signature tells you everything you need to know about how to call it.

### 4. Retry external calls, but bound them

```python
from tenacity import retry, stop_after_attempt, wait_exponential, retry_if_exception_type
import httpx

@retry(
    stop=stop_after_attempt(5),
    wait=wait_exponential(multiplier=1, min=2, max=30),
    retry=retry_if_exception_type(httpx.HTTPError),
)
def get_repo(repo: str) -> dict:
    r = httpx.get(f"https://api.github.com/repos/{repo}")
    r.raise_for_status()
    return r.json()
```

Don't retry forever. Don't retry on user error (4xx). Add jitter. Log every retry.

### 5. Configure via environment, not hard-coded values

```python
from os import environ
GITHUB_TOKEN = environ["GITHUB_TOKEN"]              # required, fail fast if missing
LOG_LEVEL = environ.get("LOG_LEVEL", "INFO")        # optional with default
```

Or use `pydantic-settings` for typed config:
```python
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    github_token: str
    log_level: str = "INFO"
    rate_limit_buffer: int = 100
```

### 6. Make output structured when run by other tools

If your script is going to be parsed by another script (or fed into a dashboard), output JSON. If it's going to be read by a human, output pretty text. Support both with a flag:

```python
if args.json:
    print(json.dumps(result))
else:
    rich.print(f"Found [green]{len(result)}[/] PRs")
```

## Patterns specific to DevOps automation

### Pattern: the "find-and-act" loop

90% of automation scripts fit this shape:

```python
def main():
    items = fetch_items()                      # find
    for item in items:
        if should_act(item):                   # filter
            try:
                act(item)                      # act
                logger.info("acted on %s", item.id)
            except Exception:
                logger.exception("failed on %s", item.id)
```

Notice: errors on individual items don't stop the loop. The dead-letter approach (log it, keep going) is almost always right for batch automation.

### Pattern: the GitHub event handler

For webhooks and bots, you often have:

```python
from fastapi import FastAPI, Header, HTTPException
import hmac, hashlib

app = FastAPI()

def verify(secret: str, signature: str, body: bytes) -> bool:
    expected = "sha256=" + hmac.new(secret.encode(), body, hashlib.sha256).hexdigest()
    return hmac.compare_digest(expected, signature)

@app.post("/webhook")
async def webhook(
    request: Request,
    x_hub_signature_256: str = Header(...),
    x_github_event: str = Header(...),
):
    body = await request.body()
    if not verify(WEBHOOK_SECRET, x_hub_signature_256, body):
        raise HTTPException(401)
    payload = json.loads(body)
    handler = HANDLERS.get(x_github_event)
    if handler:
        await handler(payload)
    return {"status": "ok"}
```

Always verify webhook signatures. Always.

### Pattern: the "run it in CI on a schedule" tool

Lots of DevOps automation isn't a server — it's a CLI that runs on a cron. Schedule it via GitHub Actions:

```yaml
on:
  schedule:
    - cron: '0 9 * * MON-FRI'
  workflow_dispatch:  # also allow manual runs
```

Inside, run your Python tool with a token. This pattern is very low-maintenance compared to running a long-lived service.

### Pattern: the "self-service" Slack bot

A common DevOps bot pattern: developers type `/redeploy staging` in Slack, and your bot kicks off a deployment. The plumbing usually looks like:

```
Slack → Webhook → small FastAPI service → triggers a GitHub Actions workflow_dispatch
```

The bot itself does almost nothing — it just translates Slack events into pipeline triggers. The actual work happens in CI, where it's auditable and parameterised.

## A worked example: the stale PR bot

Here's a complete, useful tool. Save as `stale_pr_bot.py`:

```python
"""Comments on stale pull requests in a repo."""
from __future__ import annotations
import argparse
import logging
import sys
from datetime import datetime, timedelta, timezone
from os import environ

import httpx
from tenacity import retry, stop_after_attempt, wait_exponential

logger = logging.getLogger("stale-pr-bot")

GITHUB_API = "https://api.github.com"


@retry(stop=stop_after_attempt(3), wait=wait_exponential(multiplier=1, max=10))
def github_get(client: httpx.Client, path: str) -> list[dict]:
    """GET a paginated GitHub endpoint, returns all items."""
    items: list[dict] = []
    url: str | None = f"{GITHUB_API}{path}"
    while url:
        r = client.get(url)
        r.raise_for_status()
        items.extend(r.json())
        url = r.links.get("next", {}).get("url")
    return items


def github_post(client: httpx.Client, path: str, body: dict) -> None:
    r = client.post(f"{GITHUB_API}{path}", json=body)
    r.raise_for_status()


def is_stale(pr: dict, days: int) -> bool:
    updated = datetime.fromisoformat(pr["updated_at"].replace("Z", "+00:00"))
    cutoff = datetime.now(timezone.utc) - timedelta(days=days)
    return updated < cutoff


def already_warned(pr: dict, client: httpx.Client) -> bool:
    comments = github_get(client, f"/repos/{pr['base']['repo']['full_name']}/issues/{pr['number']}/comments")
    return any("[stale-bot]" in c.get("body", "") for c in comments)


def warn(pr: dict, client: httpx.Client, dry_run: bool) -> None:
    repo = pr["base"]["repo"]["full_name"]
    msg = (
        "[stale-bot] This PR has had no activity for a while. "
        "If it's still relevant, please update it. Otherwise we'll close it in 7 days."
    )
    if dry_run:
        logger.info("would comment on %s#%d", repo, pr["number"])
        return
    github_post(client, f"/repos/{repo}/issues/{pr['number']}/comments", {"body": msg})
    logger.info("commented on %s#%d", repo, pr["number"])


def main() -> int:
    parser = argparse.ArgumentParser()
    parser.add_argument("--repo", required=True, help="owner/name")
    parser.add_argument("--days", type=int, default=14)
    parser.add_argument("--dry-run", action="store_true")
    args = parser.parse_args()

    logging.basicConfig(level=logging.INFO, format="%(asctime)s %(levelname)s %(message)s")

    token = environ["GITHUB_TOKEN"]
    headers = {"Authorization": f"Bearer {token}", "Accept": "application/vnd.github+json"}

    with httpx.Client(headers=headers, timeout=30.0) as client:
        prs = github_get(client, f"/repos/{args.repo}/pulls?state=open&per_page=100")
        logger.info("found %d open PRs", len(prs))

        acted = 0
        for pr in prs:
            try:
                if not is_stale(pr, args.days):
                    continue
                if already_warned(pr, client):
                    continue
                warn(pr, client, args.dry_run)
                acted += 1
            except httpx.HTTPError:
                logger.exception("failed on PR #%d", pr["number"])

        logger.info("acted on %d PRs", acted)
        return 0


if __name__ == "__main__":
    sys.exit(main())
```

Run as a scheduled GitHub Action:

```yaml
name: Stale PR bot
on:
  schedule:
    - cron: '0 9 * * MON'  # every Monday 9am UTC
  workflow_dispatch:
    inputs:
      dry_run:
        type: boolean
        default: true

jobs:
  run:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with: { python-version: "3.11" }
      - run: pip install httpx tenacity
      - env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |
          python stale_pr_bot.py \
            --repo ${{ github.repository }} \
            --days 14 \
            ${{ inputs.dry_run && '--dry-run' || '' }}
```

This is small enough to fit in your head and useful enough to deploy on a real repo.

## Backend services for DevOps purposes

Some specific patterns when you're writing a backend service as part of platform engineering:

### Health endpoints
- `/healthz` — am I running? (for load balancer probes)
- `/readyz` — am I ready to serve traffic? (for Kubernetes readiness probes)
- `/metrics` — Prometheus metrics

### Graceful shutdown
Listen for SIGTERM. Stop accepting new requests. Drain in-flight requests. Then exit. Kubernetes sends SIGTERM, waits 30s by default, then SIGKILLs you. If you don't handle SIGTERM, you'll drop in-flight requests on every deploy.

### Structured logging
Output JSON logs in production. They're parseable by Loki/CloudWatch/etc. Use `python-json-logger` or `structlog`.

```python
import structlog
log = structlog.get_logger()
log.info("processed_request", request_id="abc123", duration_ms=42, user_id=99)
```

### Concurrency model
For Python web services:
- I/O-bound (most things): use async (FastAPI + uvicorn) or run multiple sync workers (gunicorn).
- CPU-bound: Python's GIL hurts you. Either drop to a multiprocess model or rewrite the hot path in something else.

### Don't use Python for everything
If a task is genuinely performance-critical — high-throughput data processing, real-time systems — Python is the wrong tool. Reach for Go, Rust, or whatever fits. Recognising "this isn't a Python job" is a senior skill.

## See `lab.md` for hands-on exercises.
