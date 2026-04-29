# Module 3 — GitHub Actions: composite, reusable, and the action ecosystem

> *"Building composite GitHub Actions workflows..."*

GitHub Actions has three reuse mechanisms that look similar and get confused constantly. Knowing the difference is the single biggest "you sound senior" signal in any conversation about CI on GitHub.

## The three reuse mechanisms

### 1. Composite Actions

A **composite action** is a self-contained, packaged set of steps you call from a workflow. It lives in its own directory (or its own repo) with an `action.yml` file. From the caller's perspective, it looks like a single step.

```
my-action/
├── action.yml
└── (optionally: scripts, Dockerfile, etc.)
```

Used like:
```yaml
- uses: my-org/my-action@v1
  with:
    some-input: foo
```

**When to use composite actions:**
- You have 5–10 steps that are repeated across many workflows (setup language, install tools, configure auth, etc.).
- The logic is *contained* — it doesn't need to know about the rest of the workflow's structure.
- You want a stable interface (inputs/outputs) that hides implementation details.

### 2. Reusable Workflows

A **reusable workflow** is an entire workflow that you can call from another workflow. It's an entire YAML file in `.github/workflows/`, marked `on: workflow_call`.

Used like:
```yaml
jobs:
  call-shared-workflow:
    uses: my-org/my-repo/.github/workflows/build.yml@v1
    with:
      environment: staging
    secrets: inherit
```

**When to use reusable workflows:**
- You want to share an entire job structure (multiple jobs, dependencies between them).
- Different repos should have *the same pipeline* with parameters, not just the same steps.
- You want to enforce organisational policies — "every service must use this build workflow".

### 3. Custom JavaScript / Docker actions

A custom action that runs JavaScript (Node.js) or a Docker container. Used when shell scripting isn't enough — for example, calling complex APIs, parsing JSON elaborately, or needing libraries.

**When to use:**
- You need real programming, not shell + YAML glue.
- Performance matters (JavaScript actions start much faster than Docker actions, which spin up a container).
- You need cross-platform support (Docker actions only run on Linux runners).

### Side-by-side decision matrix

| Situation | Use |
|-----------|-----|
| "Setup our standard Python environment with our internal tools" | Composite action |
| "Every service should follow this exact build-test-publish pipeline" | Reusable workflow |
| "Parse this complex API response and return three outputs" | JavaScript action |
| "I need to run this Go CLI tool" | Composite (call the binary in shell) or Docker |

## Anatomy of a composite action

A real-world composite action that sets up a Python project. Save as `actions/setup-python-project/action.yml`:

```yaml
name: Setup Python project
description: |
  Checkout, install Python with our standard version, install dependencies
  with caching, and configure tools.
inputs:
  python-version:
    description: Python version to install
    required: false
    default: "3.11"
  requirements-files:
    description: Newline-separated list of requirements files
    required: false
    default: requirements.txt
  install-extras:
    description: Whether to install dev dependencies
    required: false
    default: "false"
outputs:
  cache-hit:
    description: Whether the dependency cache was a hit
    value: ${{ steps.setup.outputs.cache-hit }}
runs:
  using: composite
  steps:
    - name: Set up Python
      id: setup
      uses: actions/setup-python@v5
      with:
        python-version: ${{ inputs.python-version }}
        cache: pip
        cache-dependency-path: ${{ inputs.requirements-files }}

    - name: Upgrade pip
      shell: bash
      run: python -m pip install --upgrade pip

    - name: Install requirements
      shell: bash
      run: |
        for f in ${{ inputs.requirements-files }}; do
          pip install -r "$f"
        done

    - name: Install dev extras
      if: inputs.install-extras == 'true'
      shell: bash
      run: pip install -r requirements-dev.txt
```

**Things to notice:**

- Every step inside a composite action **must specify `shell:`** explicitly. Workflows have a default shell; composite actions don't.
- Inputs are typed as strings only. If you want booleans, you compare them with `== 'true'`.
- Outputs come from step outputs and need explicit wiring through `value:`.
- The `using: composite` line is what makes this a composite action.

## Anatomy of a reusable workflow

`.github/workflows/build-and-publish.yml`:

```yaml
name: Build and publish container image

on:
  workflow_call:
    inputs:
      image-name:
        required: true
        type: string
      dockerfile:
        required: false
        type: string
        default: Dockerfile
      push:
        required: false
        type: boolean
        default: true
    outputs:
      image-digest:
        value: ${{ jobs.build.outputs.digest }}
    secrets:
      registry-token:
        required: false

jobs:
  build:
    runs-on: ubuntu-latest
    outputs:
      digest: ${{ steps.build.outputs.digest }}
    permissions:
      contents: read
      packages: write
    steps:
      - uses: actions/checkout@v4
      - uses: docker/setup-buildx-action@v3
      - uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.registry-token || secrets.GITHUB_TOKEN }}
      - id: meta
        uses: docker/metadata-action@v5
        with:
          images: ghcr.io/${{ github.repository_owner }}/${{ inputs.image-name }}
          tags: |
            type=sha,format=long
            type=ref,event=branch
      - id: build
        uses: docker/build-push-action@v6
        with:
          context: .
          file: ${{ inputs.dockerfile }}
          push: ${{ inputs.push }}
          tags: ${{ steps.meta.outputs.tags }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

Called by another workflow:

```yaml
jobs:
  build:
    uses: my-org/shared-workflows/.github/workflows/build-and-publish.yml@v1
    with:
      image-name: payments-svc
    secrets: inherit
```

**`secrets: inherit`** is the most important line — it passes all secrets from the caller to the called workflow. Without it, the called workflow has no secrets and authentication will fail mysteriously.

## Composite vs reusable workflow: a worked example

Suppose you want to standardise CI across 30 microservices. You have two design options:

### Option A — Composite actions, individual workflows

Each service has its own `.github/workflows/ci.yml`, but it uses your shared composite actions:

```yaml
jobs:
  ci:
    runs-on: ubuntu-latest
    steps:
      - uses: my-org/actions/setup-python-project@v1
      - uses: my-org/actions/run-quality-gates@v1
      - uses: my-org/actions/build-and-push@v1
```

**Pros:** Each service can customise easily (skip a step, add a step). Composite actions are versioned independently.
**Cons:** No way to enforce "this exact pipeline shape". Service teams can copy the workflow once and never update it.

### Option B — Reusable workflow

Each service has a tiny `.github/workflows/ci.yml`:

```yaml
jobs:
  ci:
    uses: my-org/shared-workflows/.github/workflows/python-service-ci.yml@v1
    secrets: inherit
```

**Pros:** Total enforcement. Update the central workflow, all 30 services get the change next push.
**Cons:** Less flexibility. If a service legitimately needs a different shape, you have to either fork the workflow or add another input to the central one.

**Real-world advice:** Use both. Reusable workflows for the overall pipeline shape; composite actions for the individual building blocks inside it. That way teams *can* deviate when needed, but the path of least resistance is consistency.

## Versioning reusable components

Don't reference actions or workflows by branch name in production. `@main` means "whatever's on main right now", which means a breaking change can land on you without warning.

Three options, in order of safety:

1. **`@<commit SHA>`**: bulletproof. Locked to exactly that code. Hard to read, doesn't auto-update with security patches.
2. **`@v1` (a tag)**: nice and readable. Tags are mutable, so you have to trust the publisher not to retag.
3. **Major version float**: `@v1` where you regularly retag the latest minor/patch. Most actions in the marketplace do this. Best UX, requires trust.

For internal actions, use SHAs in the security-critical workflows (anything that touches secrets, deploys to prod, etc.). Use tags for the rest.

## Common pitfalls

### `${{ }}` evaluation timing

Expressions get evaluated at different times depending on where they appear. Inside a `run:` step they're evaluated in the runner shell context. Inside an `if:` they're evaluated by the GitHub Actions engine. This matters when secrets or variables aren't set yet.

```yaml
# ❌ This evaluates ${{ secrets.X }} during YAML parsing — secrets get logged
- run: curl -H "Authorization: ${{ secrets.X }}" ...

# ✅ This passes the secret as an env var, never appears in logs
- env:
    TOKEN: ${{ secrets.X }}
  run: curl -H "Authorization: $TOKEN" ...
```

### Permissions

By default, the `GITHUB_TOKEN` has very broad permissions. Modern best practice is to declare permissions explicitly **and minimally**, both at the workflow level and per-job:

```yaml
permissions:
  contents: read     # default for almost everything
  packages: write    # only on jobs that push images
  id-token: write    # only on jobs that need OIDC
```

Untrusted code (e.g. PRs from forks) should never run with write permissions.

### `pull_request_target` is a footgun

`pull_request_target` runs in the context of the *target* branch with full secrets, even for PRs from forks. It's used for things like "auto-label PRs", but if you accidentally run untrusted code from the PR branch under it, you've handed an attacker your secrets. Use it only for non-code operations.

### Matrix builds with `fail-fast`

By default, if one matrix entry fails, all others get cancelled (`fail-fast: true`). For tests, this is usually wrong — you want to see *all* failures, not just the first. Set `fail-fast: false` for test matrices.

## See `lab.md` for the hands-on exercise.
