# Module 4 — Lab: GitOps with FluxCD

You will install Flux into a local Kubernetes cluster, deploy a microservice through GitOps, and observe the reconciliation loop in action.

## Prerequisites

```bash
# Install kind (or use minikube)
brew install kind kubectl flux  # macOS
# or:
# curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.24.0/kind-linux-amd64
# curl -Lo ./flux https://github.com/fluxcd/flux2/releases/...

# Verify
kind version
kubectl version --client
flux --version
```

You also need a GitHub account and a personal access token with `repo` scope.

## Step 1 — Create a cluster

```bash
kind create cluster --name gitops-lab
kubectl cluster-info
```

## Step 2 — Bootstrap Flux

`flux bootstrap` does several things in one command:
- Installs Flux controllers in your cluster
- Creates a Git repo (or uses an existing one) for your cluster's state
- Configures the cluster to reconcile from that repo
- Commits Flux's own configuration to the repo (Flux manages itself!)

```bash
export GITHUB_TOKEN=ghp_yourTokenHere
export GITHUB_USER=yourusername

flux bootstrap github \
  --owner=$GITHUB_USER \
  --repository=fleet-infra \
  --branch=main \
  --path=./clusters/gitops-lab \
  --personal
```

Watch the controllers come up:
```bash
flux check
kubectl -n flux-system get pods
```

You should see four pods: `source-controller`, `kustomize-controller`, `helm-controller`, `notification-controller`.

Clone your new `fleet-infra` repo locally — that's where you'll commit changes.

## Step 3 — Add an application

Create a separate repo called `podinfo-manifests` (or use the public `stefanprodan/podinfo` example).

For this lab, let's use the public podinfo as a sample app. In your `fleet-infra` repo:

`clusters/gitops-lab/podinfo-source.yaml`:
```yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: podinfo
  namespace: flux-system
spec:
  interval: 1m
  url: https://github.com/stefanprodan/podinfo
  ref:
    branch: master
```

`clusters/gitops-lab/podinfo-kustomization.yaml`:
```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: podinfo
  namespace: flux-system
spec:
  interval: 5m
  path: "./kustomize"
  prune: true
  sourceRef:
    kind: GitRepository
    name: podinfo
  targetNamespace: default
  healthChecks:
    - apiVersion: apps/v1
      kind: Deployment
      name: podinfo
      namespace: default
  timeout: 2m
```

Commit and push:
```bash
git add clusters/gitops-lab/
git commit -m "feat: add podinfo via GitOps"
git push
```

Wait for Flux to reconcile (one minute by default):
```bash
flux get sources git
flux get kustomizations
kubectl get pods
```

You should see `podinfo` deployed without ever running `kubectl apply` yourself. **This is the GitOps loop closing.**

## Step 4 — Observe drift detection

Try to drift the cluster manually:
```bash
kubectl scale deployment podinfo --replicas=5
```

Wait up to 5 minutes. Run:
```bash
kubectl get deployment podinfo
```

The replica count should be back to whatever's in Git. Flux detected drift and reconciled.

This is the single most powerful demonstration of GitOps. Show this to a developer once and they get it.

## Step 5 — Make a real change

In your `fleet-infra` repo, add a `Kustomization` patch that overrides the replica count. Create `clusters/gitops-lab/podinfo-patch.yaml` and integrate it via Kustomize. Commit. Push.

Within a minute, your cluster reflects the change. Note that *you didn't run kubectl*. The change happened by committing to Git.

## Step 6 — Set up image automation

Now the interesting bit. We'll have Flux automatically deploy new versions of an image we control.

### 6a — Build your own image

Use the `payments-svc` from module 2's lab. Make sure it's pushing semver-tagged images to GHCR (e.g. `1.0.0`, `1.0.1`, etc).

### 6b — Create a manifests repo

`payments-manifests/base/deployment.yaml`:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payments-svc
spec:
  replicas: 1
  selector:
    matchLabels:
      app: payments-svc
  template:
    metadata:
      labels:
        app: payments-svc
    spec:
      containers:
        - name: payments-svc
          image: ghcr.io/yourname/payments-svc:1.0.0 # {"$imagepolicy": "flux-system:payments-svc"}
          ports:
            - containerPort: 8000
```

The comment after the image is critical — it's how Flux finds the image to update.

### 6c — Add the image automation resources to fleet-infra

`clusters/gitops-lab/payments-image-automation.yaml`:
```yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: payments-manifests
  namespace: flux-system
spec:
  interval: 1m
  url: https://github.com/yourname/payments-manifests
  ref:
    branch: main
  secretRef:
    name: github-deploy-key

---
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: payments
  namespace: flux-system
spec:
  interval: 2m
  path: "./base"
  prune: true
  sourceRef:
    kind: GitRepository
    name: payments-manifests
  targetNamespace: default

---
apiVersion: image.toolkit.fluxcd.io/v1beta2
kind: ImageRepository
metadata:
  name: payments-svc
  namespace: flux-system
spec:
  image: ghcr.io/yourname/payments-svc
  interval: 1m
  secretRef:
    name: ghcr-credentials

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
  interval: 2m
  sourceRef:
    kind: GitRepository
    name: payments-manifests
  git:
    checkout:
      ref:
        branch: main
    commit:
      author:
        email: fluxbot@example.com
        name: fluxbot
      messageTemplate: |
        Automatic image update
        {{range .Changed.Changes}}{{.OldValue}} -> {{.NewValue}}{{end}}
    push:
      branch: main
  update:
    path: "./base"
    strategy: Setters
```

You'll need:
- A GitHub deploy key with **write** access to `payments-manifests` (Flux needs to commit back).
- A `ghcr-credentials` secret to read from GHCR. (`flux create secret oci ghcr-credentials --url=ghcr.io --username=$GH_USER --password=$GH_TOKEN`)

### 6d — Watch the magic

Bump the version in your `payments-svc` repo (e.g. tag `v1.0.1`). The CI pipeline will build and push `ghcr.io/yourname/payments-svc:1.0.1`.

Within a couple of minutes:
1. Flux's `ImageRepository` scanner sees the new tag.
2. `ImagePolicy` evaluates: "1.0.1 matches `^1.0.0`, and it's higher than 1.0.0, so this is the latest."
3. `ImageUpdateAutomation` commits a change to the `payments-manifests` repo, updating the deployment YAML.
4. Flux's `Kustomization` controller sees the new commit and applies it.
5. The pod is recreated with the new image.

Run `kubectl get pods -w` while this happens and watch.

## Tasks

1. **Break it on purpose.** Push an image with tag `99.0.0-broken` whose container immediately crashes. What does Flux do? (Hint: look at `kubectl describe kustomization payments` — health checks should fail and Flux should report the failure but not roll back automatically. That's by design — GitOps doesn't auto-rollback; you commit a revert.)
2. **Add a staging environment.** Use Kustomize overlays. Create `environments/staging/` and `environments/production/` with different replica counts. Add two `Kustomization` resources in fleet-infra, one per environment. Deploy them to two different namespaces.
3. **Constrain image automation to staging only.** Production should require manual promotion. Set up an `ImagePolicy` that updates staging automatically but not production. Document how a human promotes from staging to prod.
4. **Add a notification.** Use Flux's `Alert` and `Provider` resources to post to Slack (or Discord/Teams) when a reconciliation fails.

## Reflection

1. Imagine you find a critical security bug in production at 2am. What's faster: GitOps revert or `kubectl rollout undo`? Why might the slower option be safer?
2. Your GitOps repo has 47 microservices and 3 environments. Is that one repo, three repos (one per env), 47 repos, or 141 (one per service per env)? Argue for one of those.
3. Drift detection in step 4 is mind-blowing the first time. But it has a cost — Flux is constantly running, constantly polling. What's the cost trade-off vs the benefit?
4. Your developer says: "But how do I deploy a hotfix quickly with GitOps? In the old system I just SSHed in and patched the running pod." Write the answer in 4 sentences.
