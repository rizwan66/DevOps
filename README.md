# DevOps Engineering Mastery Project

A structured learning project covering the eight competencies a senior DevOps / Platform Engineer is expected to own end-to-end. Each module is self-contained, has a hands-on lab, and ends with success criteria so you know when you've actually internalised it (rather than just read it).

---

## How to use this project

Work through the modules in order. Each one builds on concepts from the previous one. Budget roughly **one to two weeks per module** if you also do the lab — less if you're just reading, but reading alone won't get you there.

Each module folder contains:

- `README.md` — concept deep-dive (the "why" and "what")
- `lab.md` — hands-on exercise (the "how")
- supporting code/config files where relevant

You'll need the following installed locally to do the labs end-to-end. None of them are strictly required for reading.

- Docker Desktop (or Podman)
- `kind` or `minikube` for a local Kubernetes cluster
- `kubectl`, `helm`, `flux` CLI
- Terraform >= 1.6
- Python 3.11+
- A GitHub account (free tier is fine)
- A free-tier cloud account (AWS or GCP) for the Terraform module — optional, you can use LocalStack instead

---

## The eight modules and what they map to

| # | Module | Job-description phrase it covers |
|---|--------|----------------------------------|
| 1 | Roadmap & DevOps fundamentals | "Driving DevOps initiatives and facilitating end-to-end project workflows" |
| 2 | CI/CD/CT pipelines | "Developing and maintaining CI/CD/CT pipelines for efficient software delivery" |
| 3 | GitHub Actions composite workflows | "Building composite GitHub Actions workflows" |
| 4 | GitOps with FluxCD on Kubernetes | "Implemented GitOps-driven microservices delivery with FluxCD on Kubernetes clusters" |
| 5 | Terraform & infrastructure-as-code | "Infrastructure provisioning via Terraform for cloud-native applications" |
| 6 | Monitoring with Prometheus, Grafana, Loki | "Implemented monitoring and alerting services using Prometheus, Grafana and Loki" |
| 7 | Python backend automation | "Backend automation scripts using Python" + "scalable backend services" |
| 8 | Consulting & integration playbook | "Consulting and collaborating with running projects SW integration teams" |

---

## The mental model: how it all fits together

Before diving into individual modules, it helps to see the whole picture. A modern delivery pipeline for a cloud-native application looks roughly like this:

```
Developer pushes code
        │
        ▼
┌───────────────────┐
│  GitHub (source)  │
└─────────┬─────────┘
          │ webhook
          ▼
┌───────────────────────────────┐    ┌─────────────────────────────┐
│  GitHub Actions (CI)          │    │  Python automation scripts  │
│  - lint / test / scan         │◄──►│  - release notes            │
│  - build container image      │    │  - changelog                │
│  - push to registry           │    │  - PR triage                │
│  - update manifest repo       │    └─────────────────────────────┘
└─────────┬─────────────────────┘
          │ commit to GitOps repo
          ▼
┌───────────────────────────────┐
│  GitOps repo (manifests)      │
└─────────┬─────────────────────┘
          │ FluxCD reconciles
          ▼
┌───────────────────────────────┐    ┌─────────────────────────────┐
│  Kubernetes cluster           │◄──►│  Terraform                  │
│  (provisioned by Terraform)   │    │  - cluster, networking      │
└─────────┬─────────────────────┘    │  - IAM, databases, queues   │
          │ scrape / log / trace     └─────────────────────────────┘
          ▼
┌───────────────────────────────┐
│  Prometheus / Loki / Grafana  │
│  - metrics, logs, alerts      │
└───────────────────────────────┘
```

Every module in this project is one of those boxes or one of the arrows between them. When you finish, you should be able to draw this diagram from memory and explain what fails if any one piece is missing.

---

## Success criteria for the whole project

You're done when you can do all of the following without looking things up:

1. Stand up a `kind` cluster, install FluxCD, and have it reconcile a sample app from a Git repo you control
2. Write a Terraform module that provisions a small VPC + an EKS or GKE cluster, with remote state in S3/GCS
3. Build a composite GitHub Action that lints, tests, builds, and pushes a container image to GHCR, then bumps an image tag in a separate manifests repo
4. Write a Python script that uses the GitHub API to comment on stale PRs, deployed as a scheduled GitHub Actions workflow
5. Configure Prometheus to scrape a sample app, write a useful alerting rule, and explain why your rule won't page on transient blips
6. Diagnose a failing CI pipeline that someone else wrote, and write up the root cause and fix in a way that teaches the team rather than blames them
7. Explain "GitOps" to a developer who's never heard of it, in under three minutes, and convince them why it's better than `kubectl apply` from a laptop

If you can do those seven things, you've covered everything in the original requirements.

---

## A note on "DevOps initiatives" and "end-to-end workflows"

These phrases sound like fluff but they actually describe the hardest part of the job, which is **everything that isn't the YAML**. The YAML is the easy part. The hard parts are:

- Getting five teams with five different definitions of "done" to agree on a release process
- Convincing a senior engineer who's been hand-deploying for ten years that GitOps isn't a fad
- Knowing when to standardise (everyone uses the same base image) vs when to allow variation (each team picks their own test framework)
- Measuring whether your pipeline changes actually made delivery faster, or just made everyone feel busier

Module 8 (the consulting playbook) is dedicated entirely to this. Don't skip it. The technical modules will get you a job; module 8 will get you promoted.
