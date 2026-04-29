# Module 3 — Lab

You'll build a real, reusable GitHub Actions setup: one composite action and one reusable workflow. Then you'll consume them from a small project.

## Repo layout

You need two repos for this lab:

1. `shared-workflows` — contains your composite actions and reusable workflows.
2. `demo-service` — a minimal service that uses them.

(In a real org, the `shared-workflows` repo is often called `actions` or `platform-ci` or similar.)

## Part A — The composite action

In `shared-workflows`, create `.github/actions/quality-gates/action.yml`. This composite action runs lint + type check + tests for a Python project, with caching, in a single reusable step.

```yaml
name: Quality gates
description: Run lint, type-check and tests for a Python project
inputs:
  python-version:
    description: Python version
    required: false
    default: "3.11"
  source-dir:
    description: Directory to lint and type-check
    required: false
    default: "."
  test-command:
    description: Command to run the tests
    required: false
    default: "pytest -v"
  fail-on-coverage-below:
    description: Minimum coverage % (0 = disabled)
    required: false
    default: "0"

outputs:
  test-result:
    description: pass/fail
    value: ${{ steps.run-tests.outputs.result }}

runs:
  using: composite
  steps:
    - uses: actions/setup-python@v5
      with:
        python-version: ${{ inputs.python-version }}
        cache: pip
        cache-dependency-path: |
          requirements*.txt
          pyproject.toml

    - name: Install dependencies
      shell: bash
      run: |
        python -m pip install --upgrade pip
        pip install ruff mypy pytest pytest-cov
        if [ -f requirements-dev.txt ]; then pip install -r requirements-dev.txt
        elif [ -f requirements.txt ]; then pip install -r requirements.txt; fi

    - name: Lint
      shell: bash
      run: ruff check ${{ inputs.source-dir }}

    - name: Type check
      shell: bash
      run: mypy ${{ inputs.source-dir }} || true

    - name: Run tests
      id: run-tests
      shell: bash
      run: |
        if [ "${{ inputs.fail-on-coverage-below }}" != "0" ]; then
          ${{ inputs.test-command }} \
            --cov=${{ inputs.source-dir }} \
            --cov-fail-under=${{ inputs.fail-on-coverage-below }}
        else
          ${{ inputs.test-command }}
        fi
        echo "result=pass" >> "$GITHUB_OUTPUT"
```

Tag this repo `v1`:
```bash
git add .
git commit -m "feat: quality-gates composite action"
git tag v1
git push origin main --tags
```

## Part B — The reusable workflow

In the same `shared-workflows` repo, create `.github/workflows/python-service.yml`:

```yaml
name: Python service pipeline
on:
  workflow_call:
    inputs:
      image-name:
        required: true
        type: string
      python-version:
        required: false
        type: string
        default: "3.11"
      coverage-threshold:
        required: false
        type: string
        default: "0"
    outputs:
      image-digest:
        description: Digest of the built image
        value: ${{ jobs.build.outputs.digest }}

permissions:
  contents: read
  packages: write

jobs:
  quality:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: my-org/shared-workflows/.github/actions/quality-gates@v1
        with:
          python-version: ${{ inputs.python-version }}
          source-dir: app
          fail-on-coverage-below: ${{ inputs.coverage-threshold }}

  build:
    needs: quality
    runs-on: ubuntu-latest
    outputs:
      digest: ${{ steps.build.outputs.digest }}
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
          images: ghcr.io/${{ github.repository_owner }}/${{ inputs.image-name }}
          tags: |
            type=sha
            type=ref,event=branch
      - id: build
        uses: docker/build-push-action@v6
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

Replace `my-org/shared-workflows` with your actual org/user.

## Part C — The consumer

In `demo-service`, create a minimal app (similar to module 2). Then a tiny CI workflow:

`.github/workflows/ci.yml`:

```yaml
name: CI
on: [push, pull_request]

jobs:
  ci:
    uses: my-org/shared-workflows/.github/workflows/python-service.yml@v1
    with:
      image-name: demo-service
      coverage-threshold: "60"
    secrets: inherit
```

That's it. Three lines of caller code and you have a full pipeline.

## Tasks

### Task 1 — Get it working
Push everything. Open the Actions tab in `demo-service`. Verify the pipeline runs and uses the shared workflow. Check that you see the steps from the composite action.

### Task 2 — Make a breaking change in the composite action
Add a new required input to `quality-gates` (without a default). Push without a new tag. Run the consumer pipeline. **It should still work** because consumers are pinned to `@v1`.

Now create a new tag `v1.1.0` and update the major float (`v1` → point at `v1.1.0`). The consumer pipeline now breaks. This is the lesson of major-version floats.

### Task 3 — Add an org-wide policy
Add a step to the reusable workflow that fails if the repo doesn't have a `SECURITY.md` file. Now every project that uses your workflow is forced to have one. This is the kind of organisational policy you can enforce centrally.

### Task 4 — Output chaining
The reusable workflow already exposes `image-digest`. Extend the consumer to use that output to call a second job (e.g. update a manifest repo, post a Slack message, etc.):

```yaml
jobs:
  ci:
    uses: my-org/shared-workflows/.github/workflows/python-service.yml@v1
    # ...

  notify:
    needs: ci
    runs-on: ubuntu-latest
    steps:
      - run: echo "Built image with digest ${{ needs.ci.outputs.image-digest }}"
```

### Task 5 — Compare with a JavaScript action
Pick one composite action you wrote (or the `quality-gates` one) and try to rewrite it as a JavaScript action. Notice the friction: bundling, `node_modules`, `dist/`. When does the JavaScript approach pay off, and when is it overkill?

## Reflection

1. The composite action you wrote uses `bash`. What happens if someone tries to use it on a Windows runner?
2. If a security vulnerability is found in `actions/setup-python`, how many of your repos need updating? With composite actions, just one. Make sure you understand why this matters.
3. Imagine you have 50 services using your reusable workflow. You want to roll out a change that affects only 5 of them. How do you do that without forking the workflow? (Hint: inputs + conditional steps. Or — sometimes — `@v1.1` for the early adopters and `@v1.0` for the laggards.)
4. The reusable workflow has hard-coded `ghcr.io`. What's the cost of making the registry an input? What's the cost of keeping it hard-coded?
