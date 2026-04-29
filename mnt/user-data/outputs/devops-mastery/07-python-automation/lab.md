# Module 7 — Lab: build three real automation tools

You'll build three small Python tools that together cover most of the patterns you'll use in a DevOps role. Each is small enough to fit in your head; together they're a representative sample.

## Tool 1 — Release notes generator

A CLI that, given two Git tags, generates a markdown release notes file by querying the GitHub API for merged PRs between them.

### Spec

```
$ release-notes --repo my-org/my-repo --from v1.0.0 --to v1.1.0 > NOTES.md
```

Output:
```markdown
## Release v1.1.0

### Features
- #123 Add support for multi-region deployment (@alice)
- #145 New `--dry-run` flag (@bob)

### Bug fixes
- #134 Fix race condition in cache invalidation (@charlie)

### Other
- #129 Bump dependencies (@dependabot)
```

PRs are categorised by labels: `feature`, `bug`, otherwise "Other".

### Skeleton

```python
"""Release notes generator."""
from __future__ import annotations
import argparse, sys, logging
from os import environ
from collections import defaultdict
import httpx
from tenacity import retry, stop_after_attempt, wait_exponential

logger = logging.getLogger(__name__)
GH = "https://api.github.com"

def get_commits_between(client, repo, frm, to) -> list[dict]:
    """Use the compare API to get all commits between two refs."""
    r = client.get(f"{GH}/repos/{repo}/compare/{frm}...{to}")
    r.raise_for_status()
    return r.json()["commits"]

def find_pr_for_commit(client, repo, sha) -> dict | None:
    """Returns the PR that introduced a commit, or None."""
    r = client.get(f"{GH}/repos/{repo}/commits/{sha}/pulls")
    r.raise_for_status()
    prs = r.json()
    return prs[0] if prs else None

def categorise(pr: dict) -> str:
    labels = {l["name"] for l in pr.get("labels", [])}
    if "feature" in labels: return "Features"
    if "bug" in labels: return "Bug fixes"
    return "Other"

def main():
    # ... your implementation here
    pass
```

### Tasks
1. Implement it. Run it on a real repo (yours or a public one with tags).
2. Add a `--format json` flag for machine consumption.
3. Add it to a release workflow: when you push a `v*` tag, it auto-generates the release notes and posts them to the GitHub release.

## Tool 2 — Drift detector

A tool that periodically queries the AWS API and compares against your Terraform state to detect drift. Reports what's different.

### Spec

```
$ tf-drift --statefile s3://mycompany-tfstate/prod/network/terraform.tfstate
DRIFT detected:
  aws_subnet.public["eu-west-1a"]: 'tags' differs
    expected: {"Name": "prod-public-eu-west-1a", "Owner": "platform"}
    actual:   {"Name": "prod-public-eu-west-1a"}
```

### Hints
- `boto3` for AWS API calls
- `terraform show -json` if running locally; for CI, parse the state file from S3 directly using `boto3` to fetch it (state files are JSON).
- Compare attribute by attribute; ignore noisy fields (timestamps, etc.).

### Tasks
1. Get it working for one resource type (subnets, say).
2. Add it as a scheduled workflow that runs every hour and posts to Slack if it finds drift.
3. Reflection: is this redundant with `terraform plan`? When?

## Tool 3 — A small backend service

Build a webhook receiver that accepts GitHub `push` events and triggers a Slack message summarising the change.

### Spec

```
POST /webhook
  Headers: X-Hub-Signature-256, X-GitHub-Event
  Body: GitHub webhook payload

Behaviour:
  - Verify HMAC signature
  - For 'push' events, extract: repo, branch, commit count, author
  - POST to a Slack incoming webhook with a formatted message
  - Return 200 OK
```

### Skeleton

```python
import os, hmac, hashlib, json
from fastapi import FastAPI, Request, Header, HTTPException
import httpx

app = FastAPI()

WEBHOOK_SECRET = os.environ["GITHUB_WEBHOOK_SECRET"].encode()
SLACK_WEBHOOK = os.environ["SLACK_WEBHOOK"]

def verify(signature: str | None, body: bytes) -> bool:
    if not signature: return False
    expected = "sha256=" + hmac.new(WEBHOOK_SECRET, body, hashlib.sha256).hexdigest()
    return hmac.compare_digest(expected, signature)

@app.get("/healthz")
def healthz(): return {"status": "ok"}

@app.post("/webhook")
async def webhook(
    request: Request,
    x_hub_signature_256: str = Header(None),
    x_github_event: str = Header(None),
):
    body = await request.body()
    if not verify(x_hub_signature_256, body):
        raise HTTPException(401, "bad signature")
    if x_github_event != "push":
        return {"ignored": x_github_event}
    payload = json.loads(body)
    msg = (
        f"*{payload['pusher']['name']}* pushed "
        f"{len(payload['commits'])} commit(s) to "
        f"`{payload['repository']['full_name']}`/`{payload['ref']}`"
    )
    async with httpx.AsyncClient() as client:
        r = await client.post(SLACK_WEBHOOK, json={"text": msg})
        r.raise_for_status()
    return {"status": "ok"}
```

### Tasks
1. Run it locally. Use [`smee.io`](https://smee.io) to forward a real GitHub webhook to your localhost for testing.
2. Containerise it. Write a multi-stage Dockerfile.
3. Deploy it to your kind cluster with manifests in your GitOps repo.
4. Add Prometheus metrics: count of webhooks received by event type. Wire up a `ServiceMonitor`. Build a Grafana panel.
5. Add structured JSON logging.
6. Handle SIGTERM gracefully.

## Reflection

1. Of the three tools, which one would benefit most from being rewritten in Go? Why?
2. Tool 1 hits the GitHub API multiple times per commit. With 500 commits between releases, that's slow. How would you redesign it?
3. The webhook receiver is a "small backend service serving a large user base"-shaped problem in miniature. What changes when this service handles 10,000 webhooks per second instead of 10 per minute?
4. Pick one of your three tools and write down: what's the failure mode that wakes me up at 3am six months from now? Then add a guard against it.
