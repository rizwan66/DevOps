# Module 2 — Lab

You'll build a complete CI/CD pipeline for a small Python web service.

## What you'll build

A FastAPI service with a GitHub Actions pipeline that:

1. Runs lint (ruff), type check (mypy), and unit tests (pytest) in parallel
2. Scans for secrets (gitleaks) and vulnerable dependencies (pip-audit)
3. Builds a multi-stage Docker image with build cache
4. Pushes the image to GitHub Container Registry (GHCR)
5. Updates an image tag in a separate "manifests" repo (preview of GitOps in module 4)

## Setup

```bash
mkdir payments-svc && cd payments-svc
git init
python -m venv .venv && source .venv/bin/activate
pip install fastapi uvicorn pytest httpx ruff mypy pip-audit
```

Create the service. The code below is intentionally small — the lab is about the pipeline, not the app.

`app/main.py`:
```python
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel

app = FastAPI()

class PaymentRequest(BaseModel):
    amount: float
    currency: str

@app.get("/healthz")
def healthz() -> dict[str, str]:
    return {"status": "ok"}

@app.post("/charge")
def charge(req: PaymentRequest) -> dict[str, str]:
    if req.amount <= 0:
        raise HTTPException(status_code=400, detail="amount must be positive")
    if req.currency not in {"USD", "EUR", "GBP"}:
        raise HTTPException(status_code=400, detail="unsupported currency")
    return {"status": "charged", "id": "txn_abc123"}
```

`tests/test_main.py`:
```python
from fastapi.testclient import TestClient
from app.main import app

client = TestClient(app)

def test_healthz():
    r = client.get("/healthz")
    assert r.status_code == 200
    assert r.json() == {"status": "ok"}

def test_charge_valid():
    r = client.post("/charge", json={"amount": 10.0, "currency": "USD"})
    assert r.status_code == 200

def test_charge_negative_amount():
    r = client.post("/charge", json={"amount": -1, "currency": "USD"})
    assert r.status_code == 400

def test_charge_invalid_currency():
    r = client.post("/charge", json={"amount": 10, "currency": "XYZ"})
    assert r.status_code == 400
```

`pyproject.toml`:
```toml
[project]
name = "payments-svc"
version = "0.1.0"
requires-python = ">=3.11"

[tool.ruff]
line-length = 100

[tool.mypy]
strict = true

[tool.pytest.ini_options]
pythonpath = ["."]
```

`requirements.txt`:
```
fastapi==0.115.0
uvicorn==0.32.0
pydantic==2.9.2
```

`requirements-dev.txt`:
```
-r requirements.txt
pytest==8.3.3
httpx==0.27.2
ruff==0.7.0
mypy==1.13.0
pip-audit==2.7.3
```

## The Dockerfile (multi-stage with proper caching)

`Dockerfile`:
```dockerfile
# Stage 1: build the dependency layer separately so it caches
FROM python:3.11-slim AS deps
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir --user -r requirements.txt

# Stage 2: copy app code (rebuilds on every code change, but deps are cached)
FROM python:3.11-slim
WORKDIR /app
COPY --from=deps /root/.local /root/.local
ENV PATH=/root/.local/bin:$PATH
COPY app ./app
EXPOSE 8000
HEALTHCHECK --interval=10s --timeout=3s CMD curl -f http://localhost:8000/healthz || exit 1
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

The point of multi-stage: dependency installation (slow) is cached separately from code copying (fast). When you change app code, only the second stage rebuilds.

## The pipeline

`.github/workflows/ci.yml`:
```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:

jobs:
  # Three quality gates run in parallel
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.11"
          cache: pip
          cache-dependency-path: requirements-dev.txt
      - run: pip install -r requirements-dev.txt
      - run: ruff check .

  typecheck:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.11"
          cache: pip
          cache-dependency-path: requirements-dev.txt
      - run: pip install -r requirements-dev.txt
      - run: mypy app

  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.11"
          cache: pip
          cache-dependency-path: requirements-dev.txt
      - run: pip install -r requirements-dev.txt
      - run: pytest -v

  scan-secrets:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0  # gitleaks needs full history
      - uses: gitleaks/gitleaks-action@v2

  scan-deps:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.11"
      - run: pip install pip-audit
      - run: pip-audit -r requirements.txt

  # Image build only after all gates pass
  build:
    needs: [lint, typecheck, test, scan-secrets, scan-deps]
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    if: github.ref == 'refs/heads/main'
    steps:
      - uses: actions/checkout@v4
      - uses: docker/setup-buildx-action@v3
      - uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - id: meta
        uses: docker/metadata-action@v5
        with:
          images: ghcr.io/${{ github.repository }}
          tags: |
            type=sha,format=long
            type=raw,value=latest,enable={{is_default_branch}}
      - uses: docker/build-push-action@v6
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

## Tasks

### Task 1 — Get a baseline working

Push this to a GitHub repo. Verify all jobs run and pass. Note the total wall-clock time.

### Task 2 — Make a deliberate change and observe

Add a deliberately failing test. Push. Observe:

- Which jobs fail?
- Does the build step still run?
- How long until you got the failure feedback?

Now fix it. Push again. Note the time difference between this run and the first run (cache should help).

### Task 3 — Optimise

Pick three optimisations and implement them. Suggestions:

- Combine `lint`, `typecheck`, `test` into one job that uses a matrix — faster (one venv install) or slower (no parallelism)? Test both and decide.
- Add `paths-ignore` for documentation-only changes.
- Use `tj-actions/changed-files` to skip the build job when only test files change.

Document the before/after time for each.

### Task 4 — Image promotion

Create a second repo called `payments-svc-manifests` containing a `deployment.yaml` with an image reference. Add a job to your CI pipeline that, on push to `main`, commits an updated image tag to that manifests repo. (Hint: use a deploy key or a GitHub App token, not your personal PAT.)

This is a preview of GitOps — module 4 picks up here.

### Task 5 — Add a CT element

Add a "synthetic" job that runs every 15 minutes (cron schedule) and hits the `/healthz` endpoint of a deployed instance. Make it fail loudly if the endpoint doesn't return 200. This is the simplest possible form of continuous testing in production.

## Reflection

1. What's the bottleneck in your pipeline now? Is it a particular slow step, or coordination overhead between jobs?
2. If a developer pushes a typo in a comment, the entire pipeline runs. Is that the right behaviour, or wasteful? When would you want it different?
3. Your `build` job uses `if: github.ref == 'refs/heads/main'` — which means PRs don't get an image built. What's the trade-off there? When would you want the opposite?
4. The pipeline blocks on every test failure. When (if ever) is it correct to allow merging despite a red test?
