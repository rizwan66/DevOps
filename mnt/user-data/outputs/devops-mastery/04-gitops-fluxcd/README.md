# Module 4 — GitOps with FluxCD on Kubernetes

> *"Implemented GitOps-driven microservices delivery with FluxCD on Kubernetes clusters, increasing deployment frequency and consistency."*

This is the module where many people first realise CI/CD as they've been doing it has a fundamental problem. Read carefully.

## The fundamental problem GitOps solves

Traditional CI/CD pushes deployments. A pipeline runs `kubectl apply` (or equivalent) to make changes happen in a cluster. This has three problems that get worse at scale:

1. **Cluster credentials live in CI.** Your CI system has god-mode in your production cluster. Compromise the CI system, compromise the cluster. Every pipeline you write multiplies the attack surface.
2. **Drift is invisible.** Someone runs `kubectl edit` on Friday afternoon. Your CI pipeline's view of "what should be deployed" no longer matches reality. There's no automated reconciliation.
3. **There's no single source of truth.** If you ask "what's running in production?", the answer is "whatever the last successful pipeline deployed, plus whatever has happened since". You have to look at the cluster *and* the CI history *and* hope nobody did anything by hand.

GitOps inverts this. Instead of CI pushing changes to the cluster, an **agent inside the cluster pulls changes from Git**. The Git repository becomes the *single source of truth* for what should be running.

## The four GitOps principles (from OpenGitOps)

1. **Declarative**: System state is described in declarative files (Kubernetes manifests, Helm values, Kustomize overlays — not imperative scripts).
2. **Versioned and immutable**: The desired state is stored in Git. Git is the source of truth.
3. **Pulled automatically**: Software agents in the cluster automatically pull the desired state from Git.
4. **Continuously reconciled**: Software agents continuously observe actual state and converge it toward the desired state.

If your setup is missing any of those four, it's not GitOps; it's just CD with extra steps.

## Push CD vs Pull CD: the diagram

**Traditional Push:**
```
[CI runner] --(kubectl apply with credentials)--> [Cluster]
                            ▲
                            │
                  cluster trusts CI runner
                  (huge blast radius)
```

**Pull / GitOps:**
```
[CI runner] --(commit to Git)--> [Git repo]
                                    ▲
                                    │
                                    │ poll
                                    │
                                 [FluxCD inside cluster]
                                    │
                                    ▼
                                 [Cluster reconciles]
```

The cluster reaches outward to Git; nothing reaches inward to the cluster. This is **strictly more secure** and gives you reconciliation for free.

## Flux vs Argo CD

The two big GitOps engines for Kubernetes are FluxCD and Argo CD. Both implement the four principles. They differ in philosophy:

| | FluxCD | Argo CD |
|---|---|---|
| UI | None by default (uses kubectl + dashboards) | Rich web UI |
| Philosophy | "Just CRDs" — modular controllers | All-in-one application |
| Multi-tenancy model | Tenancy-per-controller | Project + RBAC model |
| Image automation | Built-in (`ImageRepository`, `ImagePolicy`) | Requires extra component (Argo CD Image Updater) |
| Helm support | First-class via `HelmRelease` | First-class |
| Origin | Weaveworks | Intuit |

The job description specifies FluxCD, so this module focuses on Flux. The concepts transfer almost completely to Argo CD; the YAML differs.

## FluxCD's main custom resources

Flux is composed of several controllers, each managing its own custom resources. Master these and you've mastered Flux.

### `GitRepository`

Tells Flux *where* to find your manifests.

```yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: payments-manifests
  namespace: flux-system
spec:
  interval: 1m
  url: https://github.com/my-org/payments-manifests
  ref:
    branch: main
  secretRef:
    name: github-deploy-key
```

Flux polls this every minute, fetches the latest commit, and stores the contents as a `Source` artifact other controllers can consume.

### `Kustomization`

Tells Flux *what to do* with the manifests it found. Despite the name, it works with raw manifests, Kustomize overlays, or both.

```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: payments-prod
  namespace: flux-system
spec:
  interval: 5m
  path: ./environments/production
  prune: true             # ← critical
  sourceRef:
    kind: GitRepository
    name: payments-manifests
  targetNamespace: payments
  healthChecks:
    - apiVersion: apps/v1
      kind: Deployment
      name: payments-svc
      namespace: payments
```

**`prune: true`** is the line that distinguishes GitOps from a manual `kubectl apply` loop. With pruning, if a resource is removed from Git, it's removed from the cluster. Without pruning, you accumulate orphaned resources forever.

### `HelmRelease`

For deploying Helm charts as part of your GitOps repo.

```yaml
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: redis
  namespace: payments
spec:
  interval: 10m
  chart:
    spec:
      chart: redis
      version: "20.2.x"
      sourceRef:
        kind: HelmRepository
        name: bitnami
        namespace: flux-system
  values:
    auth:
      enabled: true
    architecture: standalone
```

### `ImageRepository` and `ImagePolicy`

Flux can watch a container registry and automatically update Git when a new image tag matches a policy. This is **Flux's image automation**, and it closes the loop:

```
Developer pushes code
  → CI builds image with tag 1.2.3-abc
  → Pushes to registry
  → Flux's ImageRepository scanner sees new tag
  → ImagePolicy decides "1.2.3-abc matches semver ^1.0.0"
  → ImageUpdateAutomation commits the new tag to Git
  → Flux's Kustomization controller deploys it
```

```yaml
apiVersion: image.toolkit.fluxcd.io/v1beta2
kind: ImageRepository
metadata:
  name: payments-svc
  namespace: flux-system
spec:
  image: ghcr.io/my-org/payments-svc
  interval: 1m

---
apiVersion: image.toolkit.fluxcd.io/v1beta2
kind: ImagePolicy
metadata:
  name: payments-svc
  namespace: flux-system
spec:
  imageRepositoryRef:
    name: payments-svc
  policy:
    semver:
      range: "^1.0.0"

---
apiVersion: image.toolkit.fluxcd.io/v1beta2
kind: ImageUpdateAutomation
metadata:
  name: payments-update
  namespace: flux-system
spec:
  interval: 5m
  sourceRef:
    kind: GitRepository
    name: payments-manifests
  git:
    checkout:
      ref:
        branch: main
    commit:
      author:
        email: fluxcdbot@example.com
        name: fluxcdbot
      messageTemplate: |
        Update images
        Files: {{range .Updated.Files -}} {{.}} {{end -}}
    push:
      branch: main
  update:
    path: ./environments/production
    strategy: Setters
```

In your Kubernetes manifests, you mark the image field with a "setter" comment:

```yaml
spec:
  containers:
    - name: payments-svc
      image: ghcr.io/my-org/payments-svc:1.0.0 # {"$imagepolicy": "flux-system:payments-svc"}
```

Flux finds this comment, evaluates the policy, and updates the tag in Git when a new image matches.

## Repo structure: app code vs manifests

A common mistake is putting application code and Kubernetes manifests in the same repo. It's not strictly wrong, but it conflates two concerns:

- **App repo**: code, tests, Dockerfile. Pipeline outputs an immutable image artifact.
- **Manifests repo**: Kubernetes YAML, Helm values, Kustomize overlays. The state of your cluster.

Separation gives you:
- A clear audit trail of what was deployed (the manifests repo's git log *is* your deployment history).
- The ability to redeploy the same code with new config (just change the manifests repo).
- Cleaner permissions: developers can write to app repos, but only platform/SRE can write directly to the manifests repo.

A typical manifests repo structure:

```
payments-manifests/
├── base/                       # base Kustomize layer
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── kustomization.yaml
│   └── ingress.yaml
└── environments/
    ├── dev/
    │   ├── kustomization.yaml  # patches base
    │   └── values.yaml
    ├── staging/
    │   ├── kustomization.yaml
    │   └── values.yaml
    └── production/
        ├── kustomization.yaml
        └── values.yaml
```

Each environment is a separate `Kustomization` resource in Flux, possibly even a separate `GitRepository` if your environments live in different repos.

## Multi-cluster topology patterns

There are roughly three patterns for managing multiple clusters with GitOps:

### Hub-and-spoke
One management cluster running Flux watches a meta repo. The meta repo's manifests describe the *other* clusters. Flux on the management cluster bootstraps Flux into each spoke cluster, which then reconciles its own workloads.

### Cluster-per-environment
Each cluster runs its own Flux installation, pointed at the appropriate environment directory in the manifests repo. Simpler than hub-and-spoke but each cluster is independent.

### Single-cluster, multi-tenant
One cluster, multiple tenants/teams. Each team has its own namespace, its own `GitRepository`, its own `Kustomization`. Use Flux's tenancy features (`spec.serviceAccountName`, `spec.kubeConfig`) to scope each team's permissions.

## Anti-patterns

- **Manual `kubectl apply` "just this once"**: poisons the well. The cluster now diverges from Git. Either revert immediately or commit the change to Git.
- **Putting secrets in the GitOps repo**: encrypt them. Use SOPS, Sealed Secrets, or External Secrets Operator. Never plain text.
- **`prune: false`**: looks safer ("won't delete anything!") but accumulates orphaned resources forever and prevents real GitOps from working. Use `true` and trust your Git history.
- **Using image automation without semver discipline**: if your image tags are random hashes, image policies can't reason about them. Use semver tags or carefully designed sortable tags.
- **One giant Kustomization for everything**: when one resource is unhealthy, the whole reconciliation stalls. Split into multiple Kustomizations with `dependsOn`.

## See `lab.md` to install Flux into a local kind cluster and watch the loop close.
