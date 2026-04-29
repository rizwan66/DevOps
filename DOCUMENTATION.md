# DevOps Engineering Mastery — Full Project Documentation

> A complete reference covering all 8 modules: what each covers, how the pieces
> connect, worked examples, flowcharts, and key rules to carry into practice.

---

## Table of Contents

1. [Project Architecture — The Big Picture](#1-project-architecture--the-big-picture)
2. [Module 1 — DevOps Fundamentals & End-to-End Workflows](#2-module-1--devops-fundamentals--end-to-end-workflows)
3. [Module 2 — CI/CD/CT Pipelines](#3-module-2--cicdct-pipelines)
4. [Module 3 — GitHub Actions: Composite & Reusable Workflows](#4-module-3--github-actions-composite--reusable-workflows)
5. [Module 4 — GitOps with FluxCD on Kubernetes](#5-module-4--gitops-with-fluxcd-on-kubernetes)
6. [Module 5 — Terraform & Infrastructure as Code](#6-module-5--terraform--infrastructure-as-code)
7. [Module 6 — Monitoring with Prometheus, Grafana & Loki](#7-module-6--monitoring-with-prometheus-grafana--loki)
8. [Module 7 — Python Automation & Backend Services](#8-module-7--python-automation--backend-services)
9. [Module 8 — Consulting & SW Integration Playbook](#9-module-8--consulting--sw-integration-playbook)
10. [Cross-Module Integration Map](#10-cross-module-integration-map)
11. [Project Success Criteria Checklist](#11-project-success-criteria-checklist)

---

## 1. Project Architecture — The Big Picture

This project covers the eight competencies expected of a senior DevOps / Platform
Engineer. Every module maps to one box or one arrow in the delivery pipeline below.

```
 Developer writes code
         │
         ▼
 ┌───────────────────┐
 │   GitHub (source) │   ← Module 1 (workflow design) owns this relationship
 └─────────┬─────────┘
           │ push / webhook
           ▼
 ┌──────────────────────────────────┐    ┌───────────────────────────────┐
 │  GitHub Actions (CI)             │    │  Python automation scripts    │
 │  ├─ lint / typecheck / test      │◄──►│  ├─ stale-PR bot              │
 │  ├─ secret scan / dep scan       │    │  ├─ release-notes generator   │
 │  ├─ build container image        │    │  └─ Slack / webhook handlers  │
 │  └─ update manifests repo        │    └───────────────────────────────┘
 │                                  │      Module 7
 │  Module 2 (pipeline design)      │
 │  Module 3 (composite actions)    │
 └──────────┬───────────────────────┘
            │ commit to GitOps repo
            ▼
 ┌──────────────────────────────────┐
 │  GitOps manifests repo           │   ← Module 4 source of truth
 │  (Kubernetes YAML / Helm values) │
 └──────────┬───────────────────────┘
            │ FluxCD polls & reconciles
            ▼
 ┌──────────────────────────────────┐    ┌───────────────────────────────┐
 │  Kubernetes cluster              │◄───│  Terraform                    │
 │  (workloads, namespaces, RBAC)   │    │  ├─ VPC / subnets             │
 │                                  │    │  ├─ EKS / GKE cluster         │
 │  Module 4 (FluxCD)               │    │  ├─ IAM, databases, queues    │
 └──────────┬───────────────────────┘    │  └─ remote state (S3/GCS)     │
            │ scrape / tail logs         │                               │
            ▼                            │  Module 5                     │
 ┌──────────────────────────────────┐    └───────────────────────────────┘
 │  Prometheus / Loki / Grafana     │
 │  ├─ metrics & alerts             │
 │  ├─ log aggregation              │
 │  └─ dashboards                   │
 │                                  │
 │  Module 6                        │
 └──────────────────────────────────┘

 ← Module 8 (consulting playbook) is the horizontal layer that makes all
   the above land in real teams.
```

### How modules depend on each other

```
Module 1 ──────────────────────────────► conceptual foundation
    │
    ▼
Module 2 (pipelines) ──► Module 3 (reusable actions inside those pipelines)
    │
    ▼
Module 4 (GitOps) ◄── depends on Module 2 producing an image + manifest commit
    │
    ▼
Module 5 (Terraform) ◄── provisions the cluster that Module 4 runs on
    │
    ▼
Module 6 (Monitoring) ◄── observes the cluster provisioned in Module 5
    │
    ▼
Module 7 (Python) ◄── glue code that drives modules 2, 3, 6
    │
    ▼
Module 8 (Consulting) ◄── applies all of the above in a real team context
```

---

## 2. Module 1 — DevOps Fundamentals & End-to-End Workflows

### What it covers

The "glue" of DevOps: identifying manual handoffs, quantifying them, and replacing
them with code or explicit policy. No YAML is written in this module; instead you
build the mental model that prevents you from writing the wrong YAML.

### The DORA Four Metrics

These four numbers are the universal language for delivery performance.

```
┌──────────────────────────┬──────────────────────────────────┬──────────────┬───────────────┐
│ Metric                   │ What it measures                 │ Elite        │ Low           │
├──────────────────────────┼──────────────────────────────────┼──────────────┼───────────────┤
│ Deployment frequency     │ How often you ship to prod       │ Many/day     │ < monthly     │
│ Lead time for changes    │ Commit → production              │ < 1 day      │ > 6 months    │
│ Change failure rate      │ % of deploys causing incidents   │ 0–15%        │ 46–60%        │
│ Mean time to recovery    │ Incident detected → resolved     │ < 1 hour     │ > 1 week      │
└──────────────────────────┴──────────────────────────────────┴──────────────┴───────────────┘
```

> The metrics are coupled. Optimising one at the expense of another is a false win.

### The Seven Delivery Stages

Every software delivery workflow maps to these stages. Find the gaps to find the waste.

```
┌──────┬──────────────────┬─────────────────────────────────────────────────────────┐
│ Stage│ Name             │ Description                                             │
├──────┼──────────────────┼─────────────────────────────────────────────────────────┤
│  1   │ Plan             │ Work identified, prioritised, broken into tasks          │
│  2   │ Code             │ Developer writes the change, opens a PR                 │
│  3   │ Build            │ CI compiles/packages into an artifact                   │
│  4   │ Test             │ Automated quality gates run                             │
│  5   │ Release          │ Artifact promoted toward production                     │
│  6   │ Deploy           │ Artifact runs in production environment                 │
│  7   │ Operate          │ Service monitored; incidents detected & resolved        │
└──────┴──────────────────┴─────────────────────────────────────────────────────────┘

DevOps Engineers typically own stages 3–7. The bottleneck is usually
between stages, not within them.
```

### The Impact / Effort 2×2 — Picking the First Battle

```
              Low effort        High effort
              ──────────────────────────────
High impact │  DO FIRST     │  Plan carefully │
            ├───────────────┼─────────────────┤
Low impact  │  Quick win    │  Don't bother   │
              ──────────────────────────────
```

Always start with the top-left cell. One credible quick win gives you the
political capital to tackle the hard problem.

### The One-Page Proposal Structure

```
PROPOSAL:  [single descriptive title]
DATE:      [today]

PROBLEM    — one paragraph, quantified
PROPOSAL   — what changes
NON-GOALS  — what does NOT change (reassures skeptics)
SUCCESS    — 1–3 measurable metrics
ROLLBACK   — how to undo if it fails
TIMELINE   — rough estimate
```

### Westrum Culture Typology

Recognise which culture you are consulting into before proposing anything:

```
Pathological  → information hoarded, failure punished
               → propose blameless postmortems BEFORE GitOps

Bureaucratic  → rules over performance, new ideas create problems
               → standardise existing processes; no new tools yet

Generative    → information shared, failure = learning
               → ambitious DevOps initiatives land here
```

### Anti-Patterns to Recognise

| Anti-pattern | Why it fails |
|---|---|
| "DevOps team" as a separate silo | Recreates the Ops wall with new branding |
| Tool-driven transformation | Tools don't change how people work |
| CI without CD | Automated the easy 30%; deploys still manual |
| CD without monitoring | Ships bugs to production faster |
| Pipelines as cargo cult | Copy-pasted 14-step YAML nobody understands |

### Lab Scenario — Team Orion (Module 1 Lab)

Team Orion is a 12-person payments team with:
- 38-minute Jenkins builds triggered on push to `main`
- Manual deploy: one engineer SSHs in and runs `docker pull && docker restart`
- Production deploy every other Friday via a 23-step Confluence page
- Monitoring muted because too many false positives
- Average incident recovery: 4 hours

**Estimated DORA position:**

```
Deployment frequency  : Low  (bi-weekly)
Lead time             : Low  (days to weeks)
Change failure rate   : Medium–High (config drift)
MTTR                  : Low  (4 hours)
```

**Top bottleneck identified:** Manual deploy process (24+ eng-hours/month, medium
effort to fix, high impact) → write the one-page proposal for a scripted deploy
replacing the SSH steps.

---

## 3. Module 2 — CI/CD/CT Pipelines

### The Three Letters Clarified

```
CI  — Continuous Integration
      "Did this change break anything?"
      Every push triggers lint → test → scan → build

CD  — Continuous Delivery   → human approves production push
      Continuous Deployment → automatic push, no human

CT  — Continuous Testing
      Tests never stop: unit → integration → e2e → synthetics → chaos
```

### Reference Pipeline — Full Flowchart

```
git push
   │
   ▼
┌───────────────────────────────────────┐
│            CI  (target: ≤ 10 min)     │
│                                       │
│  ┌──────────┐  ┌──────────┐           │
│  │   lint   │  │type check│  parallel │
│  └────┬─────┘  └────┬─────┘           │
│       └──────┬───────┘                │
│  ┌───────────▼────┐  ┌────────────┐   │
│  │  unit tests    │  │secret scan │   │
│  └───────────┬────┘  └─────┬──────┘   │
│              └──────┬───────┘          │
│          ┌──────────▼─────────┐        │
│          │   build image      │        │
│          │   push to registry │        │
│          └──────────┬─────────┘        │
└─────────────────────┼──────────────────┘
                      │ image:sha-abc123
                      ▼
┌─────────────────────────────────────────┐
│  CD: dev  (auto on every passing CI)    │
│  deploy → smoke test                   │
└─────────────────┬───────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────┐
│  CD: staging  (auto on main branch)     │
│  deploy → integration → e2e → load     │
└─────────────────┬───────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────┐
│  CD: production                         │
│  canary 5% → observe → canary 50%      │
│  → observe → full rollout              │
│  (manual approval OR strong gates)     │
└─────────────────┬───────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────┐
│  CT: production  (runs forever)         │
│  synthetics + SLO monitoring + chaos   │
└─────────────────────────────────────────┘
```

### The Build-Once-Deploy-Many Rule

```
WRONG:
  build for dev → test → build for staging → test → build for prod ← bug here

RIGHT:
  build ONCE → push immutable image:sha → promote that exact image through envs
               dev → staging → prod
```

### Caching Strategy

```python
# Cache keys — what to cache on
dependency cache key  = hash(requirements.txt | package-lock.json | Cargo.lock)
layer cache           = BuildKit remote cache (registry or GHA cache)
test result cache     = pytest --lf  (last-failed only)

# What NOT to cache
- Database fixtures (leak state between builds)
- Your application's compiled output (you want fresh builds)
```

### Complete Lab Pipeline — payments-svc

```yaml
# .github/workflows/ci.yml
name: CI
on:
  push:
    branches: [main]
  pull_request:

jobs:
  lint:            # parallel
  typecheck:       # parallel
  test:            # parallel
  scan-secrets:    # parallel
  scan-deps:       # parallel

  build:
    needs: [lint, typecheck, test, scan-secrets, scan-deps]
    if: github.ref == 'refs/heads/main'
    # builds multi-stage Docker image, pushes to GHCR
```

**Multi-stage Dockerfile pattern (dependency layer cached separately):**

```dockerfile
FROM python:3.11-slim AS deps
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir --user -r requirements.txt

FROM python:3.11-slim
WORKDIR /app
COPY --from=deps /root/.local /root/.local
ENV PATH=/root/.local/bin:$PATH
COPY app ./app
EXPOSE 8000
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### Security Gates That Matter

| Gate | Tool | Action on fail |
|---|---|---|
| Secret scanning | gitleaks | Block always |
| Dependency CVEs | pip-audit / trivy | Block on HIGH/CRITICAL only |
| SAST | semgrep | Block on HIGH/CRITICAL only |
| Image signing | cosign / Sigstore | Verify at deploy time |
| SBOM generation | syft | Attach to release artifact |

### Pipeline Anti-Patterns

| Anti-pattern | Fix |
|---|---|
| Pipeline nobody reads (1200-line YAML) | Refactor to reusable workflows |
| "Always green" main branch | Tests aren't testing real behaviour |
| Manual approval with no review | Either automate the gate or make humans actually review |
| Environment-specific pipeline files | One pipeline, parameterise the environment |
| Tests that depend on order | Fix the hidden state dependency |

---

## 4. Module 3 — GitHub Actions: Composite & Reusable Workflows

### The Three Reuse Mechanisms at a Glance

```
┌──────────────────────────┬────────────────────────────────────────────────────┐
│ Mechanism                │ When to use                                        │
├──────────────────────────┼────────────────────────────────────────────────────┤
│ Composite action         │ Package 5–10 steps (setup language, install tools) │
│                          │ repeated across many workflows.                     │
├──────────────────────────┼────────────────────────────────────────────────────┤
│ Reusable workflow        │ Entire pipeline shape that multiple repos must      │
│                          │ share. Enforces structure organisation-wide.        │
├──────────────────────────┼────────────────────────────────────────────────────┤
│ JavaScript / Docker      │ Real programming logic needed (complex API calls,   │
│ action                   │ cross-platform support).                            │
└──────────────────────────┴────────────────────────────────────────────────────┘
```

### Composite Action — Anatomy

```yaml
# actions/setup-python-project/action.yml
name: Setup Python project
inputs:
  python-version:
    required: false
    default: "3.11"
outputs:
  cache-hit:
    value: ${{ steps.setup.outputs.cache-hit }}
runs:
  using: composite        # ← this is what makes it composite
  steps:
    - name: Set up Python
      id: setup
      uses: actions/setup-python@v5
      with:
        python-version: ${{ inputs.python-version }}
        cache: pip
    - name: Upgrade pip
      shell: bash          # ← every step inside composite MUST specify shell
      run: python -m pip install --upgrade pip
```

> Key rules: (1) `using: composite`, (2) every `run:` step must specify `shell:`,
> (3) inputs are strings only — compare booleans with `== 'true'`.

### Reusable Workflow — Anatomy

```yaml
# .github/workflows/build-and-publish.yml
on:
  workflow_call:
    inputs:
      image-name:
        required: true
        type: string
    outputs:
      image-digest:
        value: ${{ jobs.build.outputs.digest }}
    secrets:
      registry-token:
        required: false

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: docker/login-action@v3
        with:
          password: ${{ secrets.registry-token || secrets.GITHUB_TOKEN }}
```

Called by another workflow:

```yaml
jobs:
  build:
    uses: my-org/shared-workflows/.github/workflows/build-and-publish.yml@v1
    with:
      image-name: payments-svc
    secrets: inherit     # ← critical: without this, the called workflow has no secrets
```

### Decision Flowchart — Which Mechanism to Use?

```
"I want to share CI logic across repos"
              │
              ▼
  Do I need the same pipeline SHAPE
  (same jobs, same dependencies)?
              │
      ┌───────┴───────┐
     YES              NO
      │               │
      ▼               ▼
 Reusable         Do I need real
 Workflow         programming logic?
                       │
               ┌───────┴───────┐
              YES              NO
               │               │
               ▼               ▼
         JavaScript/        Composite
         Docker action       Action
```

### Composite + Reusable: Best-of-Both Design

```
my-org/shared-workflows/.github/workflows/python-ci.yml
   (reusable workflow — enforces the pipeline shape)
       │
       └── uses: my-org/actions/setup-python-project@v1
       └── uses: my-org/actions/run-quality-gates@v1
       └── uses: my-org/actions/build-and-push@v1
              (composite actions — implementation details)
```

Teams can deviate when needed (fork the composite action) but the path of least
resistance is consistency.

### Security Pitfalls

```yaml
# ❌ Secret in run step — gets logged
- run: curl -H "Authorization: ${{ secrets.TOKEN }}" ...

# ✅ Secret as env var — never appears in logs
- env:
    TOKEN: ${{ secrets.TOKEN }}
  run: curl -H "Authorization: $TOKEN" ...
```

**`pull_request_target` warning:** runs in the target branch context with full
secrets — even for PRs from forks. Never run untrusted PR code under it.

### Versioning Strategy

```
@<commit-SHA>   bulletproof, hard to read
@v1             readable, requires trust in publisher not to retag
@main           never use in production (anything on main breaks you)

Rule:
  security-critical workflows (deploy to prod, touch secrets) → SHA
  everything else → tag
```

---

## 5. Module 4 — GitOps with FluxCD on Kubernetes

### The Problem GitOps Solves

Traditional push-based CD has three compounding problems at scale:

```
Problem 1: CI holds cluster credentials → compromise CI = compromise cluster
Problem 2: "kubectl edit" Friday drift → real state diverges from source
Problem 3: No single source of truth → "what's in prod?" requires checking 3 systems
```

### Push CD vs Pull CD (GitOps)

```
PUSH (traditional):
  ┌───────────┐        ┌───────────┐
  │ CI runner │──creds─► Cluster  │
  └───────────┘  apply └───────────┘
  (CI has god-mode credentials; blast radius = entire cluster)

PULL (GitOps):
  ┌───────────┐        ┌──────────────┐
  │ CI runner │──push──► Git repo     │
  └───────────┘        └──────┬───────┘
                              │ poll
                              ▼
                       ┌──────────────┐
                       │ FluxCD       │  ← lives inside the cluster
                       │ (inside      │    reaches OUT to Git
                       │  cluster)    │    nothing reaches IN
                       └──────┬───────┘
                              │ apply
                              ▼
                       ┌──────────────┐
                       │   Cluster    │
                       └──────────────┘
```

The cluster reaches outward to Git. Nothing needs to reach inward.

### The Four GitOps Principles

```
1. Declarative    — desired state in declarative files (no imperative scripts)
2. Versioned      — desired state stored in Git (Git = source of truth)
3. Pulled         — agents inside the cluster pull from Git
4. Reconciled     — agents continuously converge actual → desired state
```

If any one of these four is missing, it is not GitOps.

### FluxCD Core Resources

```
GitRepository    → WHERE to find manifests (URL, branch, secret)
Kustomization    → WHAT to do with them (path, prune, health checks)
HelmRelease      → Helm charts as GitOps-managed resources
ImageRepository  → scan a container registry for new tags
ImagePolicy      → decide which tags match (semver range, alphabetical, etc.)
ImageUpdateAutomation → commit the new tag back to Git automatically
```

### The Full GitOps Loop — Sequence Diagram

```
Developer                CI                  Registry         Git Manifests    FluxCD         Cluster
   │                     │                      │                  │              │               │
   │── push code ────────►                      │                  │              │               │
   │                     │── build image ───────►                  │              │               │
   │                     │                      │                  │              │               │
   │                     │── push :1.0.1 ───────►                  │              │               │
   │                     │                      │                  │              │               │
   │                     │── commit tag update ──────────────────►  │              │               │
   │                     │                      │                  │              │               │
   │                     │                      │                  │◄─ poll ──────│               │
   │                     │                      │                  │              │               │
   │                     │                      │◄── scan ─────────│              │               │
   │                     │                      │                  │              │               │
   │                     │                      │─ new tag 1.0.1 ──►              │               │
   │                     │                      │                  │─ ImagePolicy ►              │
   │                     │                      │                  │─ commit ──────►              │
   │                     │                      │                  │              │               │
   │                     │                      │                  │◄─ reconcile ─│               │
   │                     │                      │                  │              │── apply ──────►
   │                     │                      │                  │              │               │
   │                     │                      │                  │              │         pod restarts
                                                                                             with 1.0.1
```

### Manifests Repo Structure

```
payments-manifests/
├── base/                       # Kustomize base layer
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   └── kustomization.yaml
└── environments/
    ├── dev/
    │   ├── kustomization.yaml  # patches base (e.g. 1 replica)
    │   └── values.yaml
    ├── staging/
    │   └── kustomization.yaml  # 2 replicas, staging config
    └── production/
        └── kustomization.yaml  # 3 replicas, prod config, resource limits
```

### Critical: the `prune: true` Setting

```yaml
spec:
  prune: true   # if a resource is removed from Git → removed from cluster
                # without this: cluster accumulates orphaned resources forever
                # and GitOps doesn't actually work
```

### Image Automation Marker in Manifests

```yaml
# In your deployment.yaml — the comment is parsed by Flux
containers:
  - name: payments-svc
    image: ghcr.io/my-org/payments-svc:1.0.0 # {"$imagepolicy": "flux-system:payments-svc"}
```

### Anti-Patterns

| Anti-pattern | Consequence |
|---|---|
| Manual `kubectl apply` "just this once" | Drift from Git; reconciler fights you |
| Plain-text secrets in GitOps repo | Secrets in Git = breach waiting to happen. Use SOPS/Sealed Secrets |
| `prune: false` | Orphaned resources pile up; real GitOps doesn't work |
| One giant Kustomization | One unhealthy resource stalls all reconciliation |
| Random hash image tags | ImagePolicy can't reason about ordering |

---

## 6. Module 5 — Terraform & Infrastructure as Code

### The Core Mental Model — Three Things to Keep Separate

```
┌─────────────────────┐    plan    ┌─────────────────────┐
│  Configuration      │ ◄────────► │  State file         │
│  (.tf files)        │            │  (terraform.tfstate) │
│  "what you want"    │            │  "what Terraform     │
└─────────────────────┘            │   thinks exists"     │
                                   └──────────┬──────────┘
                                              │ apply
                                              ▼
                                   ┌─────────────────────┐
                                   │  Real World          │
                                   │  (cloud resources)   │
                                   └─────────────────────┘

terraform plan   = config diff state     → shows changes
terraform apply  = real world ← state   → makes changes + updates state
```

### Remote State — Three Non-Negotiable Rules

```
1. Use remote state with locking (never local .tfstate in a team)
2. Encrypt state at rest (S3 + SSE-KMS or GCS + CMEK)
3. Restrict access (state contains secrets; treat like prod credentials)
```

**AWS backend configuration:**

```hcl
terraform {
  backend "s3" {
    bucket         = "mycompany-tfstate"
    key            = "prod/network/terraform.tfstate"
    region         = "eu-west-1"
    dynamodb_table = "mycompany-tfstate-locks"   # state locking
    encrypt        = true
    kms_key_id     = "alias/terraform-state"
  }
}
```

### State File Separation — Blast Radius Control

```
mycompany-tfstate/
├── prod/
│   ├── network/terraform.tfstate      # VPC, subnets — rarely changed
│   ├── eks/terraform.tfstate          # cluster
│   ├── databases/terraform.tfstate    # RDS, ElastiCache
│   └── apps/terraform.tfstate        # application resources
├── staging/
└── dev/
```

Smaller state files = smaller blast radius if a state gets corrupted or
accidentally applied.

### Module Structure

```
modules/eks-cluster/
├── README.md          # usage examples (required)
├── versions.tf        # Terraform + provider version constraints
├── variables.tf       # inputs (the public API)
├── outputs.tf         # outputs
├── main.tf            # primary resources
├── iam.tf             # IAM-specific
├── networking.tf      # subnets, security groups
└── examples/
    └── basic/main.tf
```

**Good module interface — grouped inputs:**

```hcl
# Instead of 25 flat variables, group related inputs as objects
variable "network" {
  type = object({
    vpc_id     = string
    subnet_ids = list(string)
  })
}

variable "compute" {
  type = object({
    instance_type    = string
    desired_capacity = number
    max_capacity     = number
  })
}
```

### Directory Layout — Environments as Directories (Not Workspaces)

```
infrastructure/
├── modules/              # generic, reusable
│   ├── eks-cluster/
│   ├── rds/
│   └── vpc/
└── environments/
    ├── dev/
    │   ├── main.tf       # calls modules with dev values
    │   ├── backend.tf    # dev-specific remote state config
    │   └── terraform.tfvars
    ├── staging/
    └── prod/

Rule: Terraform workspaces ≠ environments.
      Use separate directories per long-lived environment.
      Workspaces are for short-lived ephemeral environments only.
```

### CI/CD for Terraform — Pull-Request Gate

```
Developer opens PR with .tf changes
         │
         ▼
CI runs:  terraform plan
         │ posts plan as PR comment
         ▼
Reviewer reads the plan  ← this is the security gate
         │
         ▼
PR merged to main
         │
         ▼
CI on main runs: terraform apply
```

Tools: Atlantis (open source), Terraform Cloud, Spacelift, env0.
Do NOT run `terraform apply` from a developer's laptop in production.

### Common Pitfalls

| Pitfall | Safe alternative |
|---|---|
| `count` for collections | `for_each` — uses stable keys, not indices |
| `null_resource` + `local-exec` | Find a real resource or use a different tool |
| Hard-coded secrets in `.tf` | Use `aws_secretsmanager_secret` or env vars |
| Cross-state `terraform_remote_state` | SSM Parameter Store / Secret Manager |
| Refactoring without `moved {}` | `moved {}` blocks prevent destroy/recreate |
| Terraform workspaces for environments | Separate directories with separate backends |

---

## 7. Module 6 — Monitoring with Prometheus, Grafana & Loki

### The Three Pillars

```
┌──────────────┬──────────────────────────────────┬───────────────────────────┐
│ Pillar       │ Best for                         │ Do NOT use for            │
├──────────────┼──────────────────────────────────┼───────────────────────────┤
│ Metrics      │ Trends, alerts, dashboards       │ Debugging individual reqs │
│ (Prometheus) │ "Is the system healthy right now?"│                           │
├──────────────┼──────────────────────────────────┼───────────────────────────┤
│ Logs         │ Debugging specific events         │ High-frequency alerting   │
│ (Loki)       │ "What did request xyz123 do?"     │                           │
├──────────────┼──────────────────────────────────┼───────────────────────────┤
│ Traces       │ Latency across distributed services│ High-volume monitoring   │
│ (Tempo)      │ "Which service is the bottleneck?" │                          │
└──────────────┴──────────────────────────────────┴───────────────────────────┘
```

### Prometheus Architecture — Pull Model

```
┌──────────────────────────────────────────────────────────────┐
│  Kubernetes cluster                                          │
│                                                              │
│  ┌──────────────┐   /metrics   ┌─────────────────────────┐  │
│  │  payments-svc │◄────────────│   Prometheus             │  │
│  └──────────────┘   scrape     │   (pull model)           │  │
│                                │                          │  │
│  ┌──────────────┐   /metrics   │   ServiceMonitor CRD     │  │
│  │  auth-svc    │◄─────────────│   discovers targets      │  │
│  └──────────────┘              │   automatically          │  │
│                                └──────────┬──────────────┘  │
│                                           │ PromQL query     │
└───────────────────────────────────────────┼──────────────────┘
                                            │
                              ┌─────────────▼───────────────┐
                              │  Grafana                    │
                              │  dashboards & alerts        │
                              └─────────────────────────────┘
```

### Prometheus Metric Types

```
Counter   → only increases (or resets to 0 on restart)
            use for: total requests, total errors
            WRONG: using a gauge for totals then trying to compute rate()

Gauge     → goes up and down
            use for: memory, active connections, queue depth

Histogram → distribution of values in predefined buckets
            use for: request latency (p50/p95/p99)

Summary   → like histogram but client-side quantiles (less common)
```

### Essential PromQL Recipes

```promql
# Rate of requests per second (5-minute window)
rate(http_requests_total[5m])

# 95th-percentile response time
histogram_quantile(0.95,
  sum by (le) (rate(http_request_duration_seconds_bucket[5m])))

# Error rate
sum(rate(http_requests_total{status=~"5.."}[5m]))
  / sum(rate(http_requests_total[5m]))

# CPU usage per pod
sum by (pod) (rate(container_cpu_usage_seconds_total[5m]))

# Memory as % of limit
sum by (pod) (container_memory_working_set_bytes)
  / sum by (pod) (kube_pod_container_resource_limits{resource="memory"})
```

### Alerting Rule — Anatomy

```yaml
groups:
  - name: http_alerts
    rules:
      - alert: HighErrorRate
        expr: |
          sum(rate(http_requests_total{status=~"5.."}[5m])) by (service)
          / sum(rate(http_requests_total[5m])) by (service)
          > 0.05
        for: 10m          # ← fire only if condition holds for 10 continuous minutes
        labels:           # without "for:", every transient blip pages on-call
          severity: page
        annotations:
          summary: "{{ $labels.service }} error rate too high"
          description: "{{ $value | humanizePercentage }} errors for >10m"
          runbook: "https://wiki.internal/runbooks/high-error-rate"
```

### SLO-Based Burn-Rate Alerting

```promql
# Alert when consuming 30-day error budget at a rate that exhausts it in 1 hour
(
  sum(rate(http_requests_total{status=~"5.."}[1h])) by (service)
  / sum(rate(http_requests_total[1h])) by (service)
) > 14.4 * (1 - 0.999)

# 14.4 = "1 hour / (30 days × (1 - SLO))" from Google SRE workbook
# This fires meaningfully early without crying wolf on transient blips
```

### Dashboard Design — RED Method

Every service should have a RED dashboard:

```
Rate     — requests/second (total traffic volume)
Errors   — errors/second (broken down by type)
Duration — latency distribution (p50, p95, p99)
```

For infrastructure use USE: Utilisation, Saturation, Errors.

### Loki — Label Cardinality Rule

```
Good labels (bounded, low cardinality):
  namespace, app, container, level, env

Bad labels (unbounded = performance disaster):
  user_id, request_id, ip_address
  → each unique value creates a new log stream
  → 1 million users = 1 million streams

Rule: if a label has > ~10,000 unique values,
      it does NOT belong as a Loki label.
      Put it in the log line, filter with LogQL.
```

**LogQL example:**

```logql
{namespace="payments", container="api"}
  |= "ERROR"
  | json
  | duration > 500ms
```

### Alerting Rules That Don't Get Muted

```
1. Alert on symptoms, not causes
   BAD:  "auth pod restarted"
   GOOD: "users cannot log in"

2. Every alert must be actionable
   If on-call answer is "wait and see" → delete the alert

3. Severity tiers
   page    → wake me up NOW (≤ 5 per week max)
   warning → look at it tomorrow
   info    → context only

4. Link to runbooks in every annotation

5. Use SLO-based alerting — alert on error budget burn rate,
   not raw thresholds
```

---

## 8. Module 7 — Python Automation & Backend Services

### The DevOps Python Stack

```
HTTP / API        httpx (modern), tenacity (retries), pydantic (validation)
CLI               typer or click, rich (output)
Web services      FastAPI + uvicorn (async), gunicorn (production workers)
Cloud SDKs        boto3 (AWS), google-cloud-* (GCP), kubernetes (K8s client)
Quality           ruff (lint+format), mypy/pyright (types), pytest
Packaging         uv (modern, fast), poetry, or plain pip
```

### Six Tenets of Good DevOps Python

**1. Idempotent — safe to run twice**

```python
# Bad
def add_user_to_team(team, user):
    api.add_member(team, user)        # explodes if user already exists

# Good
def add_user_to_team(team, user):
    if user not in api.get_members(team):
        api.add_member(team, user)
```

**2. Fail loudly — never swallow exceptions**

```python
try:
    do_thing()
except SpecificExpectedError as e:
    logger.warning("expected: %s, continuing", e)
except Exception:
    logger.exception("unexpected error")   # includes traceback
    raise                                  # re-raise; don't swallow
```

**3. Type hints everywhere**

```python
def fetch_pull_requests(
    repo: str,
    state: Literal["open", "closed", "all"] = "open",
    limit: int | None = None,
) -> list[PullRequest]:
    ...
```

**4. Retry external calls with bounded backoff**

```python
from tenacity import retry, stop_after_attempt, wait_exponential

@retry(
    stop=stop_after_attempt(5),
    wait=wait_exponential(multiplier=1, min=2, max=30),
)
def get_repo(client: httpx.Client, repo: str) -> dict:
    r = client.get(f"https://api.github.com/repos/{repo}")
    r.raise_for_status()
    return r.json()
```

**5. Configure via environment**

```python
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    github_token: str          # required — fails fast if missing
    log_level: str = "INFO"    # optional with default
    rate_limit_buffer: int = 100
```

**6. Structured output in production**

```python
if args.json:
    print(json.dumps(result))    # machine-readable
else:
    rich.print(f"Found [green]{len(result)}[/] PRs")   # human-readable
```

### The "Find-and-Act" Loop Pattern

90% of automation scripts fit this shape:

```python
def main():
    items = fetch_items()                      # find everything
    for item in items:
        if should_act(item):                   # filter
            try:
                act(item)                      # act
                logger.info("acted on %s", item.id)
            except Exception:
                logger.exception("failed on %s", item.id)
                # errors on individual items do NOT stop the loop
```

### Complete Example — Stale PR Bot

```python
"""Comments on stale pull requests."""
from __future__ import annotations
import argparse, logging, sys
from datetime import datetime, timedelta, timezone
from os import environ

import httpx
from tenacity import retry, stop_after_attempt, wait_exponential

logger = logging.getLogger("stale-pr-bot")

@retry(stop=stop_after_attempt(3), wait=wait_exponential(multiplier=1, max=10))
def github_get(client: httpx.Client, path: str) -> list[dict]:
    """GET a paginated GitHub endpoint, returns all items."""
    items: list[dict] = []
    url: str | None = f"https://api.github.com{path}"
    while url:
        r = client.get(url)
        r.raise_for_status()
        items.extend(r.json())
        url = r.links.get("next", {}).get("url")
    return items

def is_stale(pr: dict, days: int) -> bool:
    updated = datetime.fromisoformat(pr["updated_at"].replace("Z", "+00:00"))
    return updated < datetime.now(timezone.utc) - timedelta(days=days)

def main() -> int:
    parser = argparse.ArgumentParser()
    parser.add_argument("--repo", required=True)
    parser.add_argument("--days", type=int, default=14)
    parser.add_argument("--dry-run", action="store_true")
    args = parser.parse_args()

    logging.basicConfig(level=logging.INFO, format="%(asctime)s %(levelname)s %(message)s")

    with httpx.Client(
        headers={"Authorization": f"Bearer {environ['GITHUB_TOKEN']}"},
        timeout=30.0,
    ) as client:
        prs = github_get(client, f"/repos/{args.repo}/pulls?state=open&per_page=100")
        for pr in prs:
            try:
                if is_stale(pr, args.days):
                    if not args.dry_run:
                        client.post(
                            f"https://api.github.com/repos/{args.repo}/issues/{pr['number']}/comments",
                            json={"body": "[stale-bot] This PR has had no activity for a while."}
                        ).raise_for_status()
            except httpx.HTTPError:
                logger.exception("failed on PR #%d", pr["number"])
    return 0

if __name__ == "__main__":
    sys.exit(main())
```

**Scheduled as a GitHub Actions workflow:**

```yaml
on:
  schedule:
    - cron: '0 9 * * MON'    # every Monday 9am UTC
  workflow_dispatch:
    inputs:
      dry_run: { type: boolean, default: true }

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
        run: python stale_pr_bot.py --repo ${{ github.repository }} --days 14
```

### Backend Service Checklist

```
Health endpoints:
  GET /healthz  → "am I running?" (liveness probe)
  GET /readyz   → "am I ready?" (readiness probe)
  GET /metrics  → Prometheus scrape endpoint

Graceful shutdown:
  Handle SIGTERM → drain in-flight requests → exit
  (K8s sends SIGTERM, waits 30s, then SIGKILLs)

Structured logs in production:
  JSON format, parseable by Loki/CloudWatch
  Use structlog or python-json-logger

Webhook security (always):
  Verify HMAC-SHA256 signature on every incoming webhook
```

### The Self-Service Bot Architecture

```
Developer types /redeploy staging in Slack
           │
           ▼
   small FastAPI service (verify Slack signature)
           │
           ▼
   triggers GitHub Actions workflow_dispatch
           │
           ▼
   pipeline runs the actual deployment (auditable, parameterised)
```

The bot does almost nothing. All real work stays in CI where it's logged.

---

## 9. Module 8 — Consulting & SW Integration Playbook

### The First Week Rule

```
Day 1–2:  shadow people doing real work — don't talk, just take notes
Day 3–4:  read CI configs, runbooks, last 6 months of postmortems, Slack history
Day 5:    30-min 1-on-1s with 4–6 people, three questions:
            1. "If you could change one thing about how we ship, what would it be?"
            2. "What's the most frustrating part of your week?"
            3. "What works well that I shouldn't break?"  ← most important

End of week 1: send a written brief (observations, opportunities, next step, non-goals)
```

### Consulting Engagement Phases

```
Week 1:   Measure (establish before-numbers; can't claim improvement without them)
          Instrument pipeline durations, flake rate, lead time, DORA baseline

Week 2–3: Quick wins
          Add caching, parallelise independent jobs, remove dead steps,
          fix top 3 flaky tests → expect 2–3× speedup without disruption
          → now you have credibility

Week 4–6: Standardise
          Convert per-repo YAML to shared reusable workflow
          Centralise secret management
          Expect pushback — address reasons, don't override them

Month 2–3: Fundamental shifts
          GitOps, cross-team platform, runtime upgrades
          These are 6-month programmes; be honest about that
```

### Diagnostic Checklist for CI/CD Pipelines

```
Pipeline duration & feedback loop
  □ Total pipeline time from push to green?  (>15 min → context switch problem)
  □ Flake rate? (>5% → developers ignore failures)
  □ Time from failure to developer notification?

Deploy mechanics
  □ Who can deploy? How many people? (< 3 = bus factor; > 10 = no standards)
  □ How do you know a deploy succeeded?
  □ Rollback tested in last 6 months? (untested = broken)

Test pyramid shape
  □ Unit : integration : e2e ratio?  (should be many : some : few)
  □ When did you last delete a test?

Configuration management
  □ Where do env-specific values live? Are they in 3 different systems?
  □ Does anyone clean up stale config?

Knowledge concentration
  □ How many "ask Marta" steps in the deploy process?
  □ What happens when the lead engineer is on holiday?
  □ Are runbooks current? (last edit date?)

Observability
  □ Can you tell in real time that a deploy made things worse?
  □ Is the alerts channel muted?  (muted = team gave up; worse than no alerts)
```

### Handling Pushback

```
"Skip tests for this hotfix"
"Hard-code prod credentials just for now"
"Disable the security scan — we need to ship Friday"

Option A — Comply + document the risk
  Do it, but write the risk in an email to the requester.
  Creates a paper trail. Not the default option.

Option B — Negotiate a smaller version
  "I can give you a one-day exception with auto-revert"
  Bend without breaking.

Option C — Refuse  ← reserve for genuine disasters
  "I won't hard-code prod credentials because [specific breach scenario].
   Here's the alternative that solves your actual need."

Order: default to A or B; save C for things that genuinely endanger the system.
Senior consultants who refuse everything stop getting invited to important meetings.
```

### Key Templates

**One-page proposal:**
```
PROBLEM      — one paragraph, quantified
PROPOSAL     — one paragraph describing the change
NON-GOALS    — what this does NOT change
SUCCESS      — 1–3 specific measurable metrics
ROLLBACK     — what to do if it fails
TIMELINE     — rough estimate
```

**Blameless postmortem:**
```
IMPACT       — duration, users affected
TIMELINE     — bullet list with timestamps (just facts)
ROOT CAUSE   — the SYSTEM reason this happened (not the person)
WHAT WENT WELL
WHAT WENT POORLY
ACTION ITEMS — specific, owned, dated
```

**Handoff document:**
```
WHAT IT IS   — one paragraph
OWNER NOW    — names
WHERE THINGS LIVE — code, runbook, dashboard, alerts (all linked)
HOW TO CHANGE IT  — normal change step-by-step
HOW TO ROLLBACK   — bad day step-by-step
WHO TO ASK   — you for 1 month; then team chat
KNOWN ISSUES — imperfect things not worth fixing now
```

### Communication Patterns That Work

| Pattern | Use when |
|---|---|
| "Here's what I'm seeing / thinking / proposing" | Any recommendation |
| "Stupid question, but..." | You don't understand something |
| "What problem are we trying to solve?" | Meeting proposes solutions before agreeing on the problem |
| Never say "easy" | Always underestimates; destroys trust when you're late |

### Knowledge Transfer — The Part Everyone Skips

The whole point of consulting is you eventually leave. Success = the team can
maintain what you built without you.

```
Pair, don't solo       → whoever inherits it should have committed code with you
Write the runbook      → not "how it works" but "what to do when it breaks"
Explicit handoff       → demo session where THEY explain it back to YOU
Defined exit date      → don't drift away; close the engagement formally
```

---

## 10. Cross-Module Integration Map

How all the pieces wire together in a single deployment of a real change:

```
1. Developer opens PR on payments-svc
   Module 1: PR review process, DORA lead-time clock starts

2. GitHub Actions CI fires (Module 2 + Module 3)
   ├─ ruff lint (composite action)
   ├─ mypy typecheck (composite action)
   ├─ pytest (composite action)
   ├─ gitleaks secret scan
   ├─ pip-audit dependency scan
   └─ docker build + push :sha-abc123 to GHCR

3. PR merged to main
   CI build job pushes image to GHCR
   Python script (Module 7) posts release notes to Slack

4. CI updates payments-manifests repo (Module 4 prep)
   Commits: image: ghcr.io/.../payments-svc:sha-abc123

5. FluxCD (Module 4) sees the commit within 1 minute
   ImagePolicy evaluates new tag
   Kustomization controller applies manifests to dev namespace
   Health check passes → staging promotion

6. Terraform (Module 5) provisioned the underlying cluster
   VPC, EKS nodes, IAM roles, RDS — all reproducible
   Separate state files per environment

7. Prometheus + Grafana + Loki (Module 6) observe the deploy
   Error rate dashboard shows no regression
   SLO burn rate alert stays silent
   Loki shows no new ERROR log patterns

8. Module 8 (consulting lens):
   Someone on the team says "can we just kubectl apply this hotfix?"
   Response: "Fastest way is to commit to Git — Flux deploys in 60s."
   "If it's an emergency: kubectl apply now, commit 5 minutes later."
```

---

## 11. Project Success Criteria Checklist

You are done when you can do all of the following without looking anything up:

```
□  Stand up a kind cluster, install FluxCD, reconcile a sample app from a
   Git repo you control

□  Write a Terraform module for a small VPC + EKS/GKE cluster with remote
   state in S3/GCS

□  Build a composite GitHub Action that lints, tests, builds, and pushes a
   container image to GHCR, then bumps an image tag in a separate manifests repo

□  Write a Python stale-PR bot deployed as a scheduled GitHub Actions workflow

□  Configure Prometheus to scrape a sample app, write a useful alerting rule,
   and explain why the `for:` duration prevents transient-blip pages

□  Diagnose a failing CI pipeline someone else wrote; produce a root-cause writeup
   that teaches rather than blames

□  Explain GitOps to a developer in under 3 minutes and convince them it is safer
   than `kubectl apply` from a laptop
```

---

*Last updated: 2026-04-29 | Branch: main*
