# Module 2 — CI / CD / CT pipelines

> *"Designing and implementing automation solutions for development and integration processes... developing and maintaining CI/CD/CT pipelines for efficient software delivery."*

This module covers the three letters that get conflated constantly. Get the distinction clear in your head and you'll instantly sound more senior than 80% of people who claim CI/CD on their CV.

## The three things

### CI — Continuous Integration

The practice of merging every developer's work into a shared mainline **multiple times per day**, and verifying each merge automatically. The point is to find integration bugs within minutes of them being introduced, when the change is small and the author still remembers what they were doing.

CI's job is to answer one question: *"Did this change break anything?"*

**A real CI pipeline runs on every push and includes:**

- Static analysis (linting, type checking)
- Unit tests
- Integration tests against fakes/test doubles
- Security scanning (SAST, dependency scanning, secret scanning)
- Build and packaging into an artifact (e.g. a container image)

A CI pipeline that takes more than 10 minutes will be ignored by developers. They'll context-switch and forget about it. Optimise ruthlessly for speed: parallelise, cache aggressively, run only the tests affected by the change when possible.

### CD — Continuous Delivery (or Deployment)

Two different things sharing the same acronym. Be precise:

- **Continuous Delivery**: every change that passes CI is automatically *prepared* for production deployment. A human still pushes the button to release.
- **Continuous Deployment**: every change that passes CI is automatically *deployed* to production. No human in the loop.

Most regulated industries (banks, healthcare) do continuous delivery. Most consumer SaaS companies do continuous deployment. Either is a valid choice; the wrong choice is doing neither and calling your fortnightly Friday-afternoon ritual "CD".

CD's job is to answer: *"Is this change safe to release?"*

**A real CD pipeline includes:**

- Promotion through environments (dev → staging → prod)
- Smoke tests in each environment after deployment
- Progressive rollout (canary, blue-green, or rolling)
- Automated rollback if health checks fail
- Audit trail (who deployed what, when, with what version)

### CT — Continuous Testing

The least standardised of the three terms. Different organisations mean slightly different things, but the core idea is: **testing is not a phase, it's a continuous activity that happens at every stage of the pipeline and continues into production**.

CT includes everything CI does, plus:

- **Contract testing** between microservices (e.g. Pact)
- **End-to-end tests** in staging environments
- **Performance and load tests** before production rollout
- **Chaos engineering** in production (controlled failure injection)
- **Synthetic monitoring** in production (continuously running test traffic)
- **Production validation** (does the new version actually behave correctly with real traffic?)

CT's mental model: tests don't end when you push to main. The pipeline keeps testing the system, indefinitely, with increasingly realistic conditions.

## The pipeline as a quality gate

A pipeline is a series of gates. Each gate has a question, and the change either passes or doesn't. Design your gates so that:

1. **Fast, cheap gates run first.** Lint before unit tests, unit tests before integration tests, integration tests before e2e tests. Failing fast saves expensive compute.
2. **Each gate has clear ownership.** When a gate fails, it should be immediately obvious whose problem it is.
3. **Gates are parallel where possible.** Running 4 things in parallel is 4× faster than serially.
4. **Failures are actionable.** "Test failed" is not actionable. "Test `test_payment_validation` failed because `assert balance == 100` got 99.99 — check rounding logic in `payments.py:47`" is actionable.

## Pipeline patterns by repository structure

### Monorepo (one repo, many services)

Trigger only the parts of the pipeline affected by the change. Use path filters or tools like Bazel/Nx/Turborepo to compute the change graph.

```yaml
# Pseudo-config showing the principle
on:
  push:
    paths:
      - 'services/payments/**'
      - 'libs/shared/**'
jobs:
  test-payments:
    if: contains(changed_paths, 'services/payments')
```

The naïve "always run everything" approach turns a 10-minute pipeline into a 90-minute one as the monorepo grows.

### Polyrepo (one repo per service)

Each repo has its own pipeline. The challenge is consistency: when you have 40 repos, you don't want to update the pipeline in 40 places when something changes. Solutions:

- **Reusable workflows** (GitHub Actions) or templates (GitLab CI)
- **Composite actions** for repeated steps
- **A platform repo** that publishes shared CI components

### The build-once-deploy-many principle

**Critical rule:** the artifact you test in CI is the *exact* artifact you deploy to production. Don't rebuild between environments. Build the container image once, push it to a registry, and promote that immutable image through environments by changing only configuration.

If you rebuild for each environment, you've created a new "compile to staging" and "compile to production" step that hasn't been tested. That's where the bugs hide.

## Caching: the difference between a good and great pipeline

Caching is the single highest-leverage optimisation you can make. A pipeline that takes 25 minutes uncached often takes 4 minutes cached. What to cache:

- **Dependency directories**: `node_modules`, `.venv`, `~/.gradle`, `~/.m2`. Key on the lockfile hash.
- **Compiled artifacts**: build outputs, type-checker results.
- **Container layers**: use BuildKit and a remote cache (e.g. GitHub's `gha` cache, registry-based cache, or `s3` cache).
- **Test results**: tools like `pytest --lf` only re-run failing tests; some CI systems support test caching natively.

What *not* to cache:

- Anything mutable that could leak state between builds (e.g. a database fixture).
- Build artifacts that are part of your output (you want fresh builds of your code, not cached ones — only cache the *inputs* to the build).

## Security gates that actually matter

Most pipelines have a "security scan" step that everyone ignores because it produces 4,000 medium-severity warnings. To make security gates effective:

- **Block on high/critical only.** Track everything else, but don't block.
- **Allow temporary exceptions with expiry dates.** Anything ignored forever will eventually bite.
- **Scan secrets on every commit.** Use `gitleaks` or similar — leaked credentials are the #1 cause of cloud breaches.
- **Sign your artifacts.** Use cosign or Sigstore to sign container images, then verify signatures at deploy time.
- **Generate an SBOM** (Software Bill of Materials) per build. When the next Log4Shell happens, you want to know within minutes which of your services is affected.

## Anti-patterns

- **Pipelines that nobody reads.** A 1,200-line GitHub Actions YAML with copy-pasted blocks is a maintenance nightmare. Refactor into reusable workflows or composite actions.
- **The "always green" pipeline.** If your `main` branch never goes red, your tests aren't testing anything real. A small percentage of red builds is healthy.
- **Manual approval steps with no review.** "Click here to deploy to prod" → click → done. The approval was theatre. Either the approver actually reviewed something, or the gate should be automated.
- **Environment-specific pipelines.** Separate `deploy-to-staging.yml` and `deploy-to-prod.yml` files diverge over time. Keep one pipeline and parameterise the environment.
- **Tests that depend on test order.** A flaky test that passes in isolation but fails in CI is usually a hidden state dependency. Fix it; don't retry it.

## A reference pipeline (conceptual)

```
┌──────────────┐
│ git push     │
└──────┬───────┘
       │
       ▼
┌─────────────────┐
│ CI              │  ◄── 8–10 min budget
│ ├─ lint         │
│ ├─ type check   │  parallel
│ ├─ unit test    │
│ ├─ scan secrets │
│ ├─ scan deps    │
│ └─ build image  │
└──────┬──────────┘
       │ image:abc123 pushed to registry
       ▼
┌─────────────────┐
│ CD: dev         │  ◄── automatic, on every passing CI
│ ├─ deploy       │
│ └─ smoke test   │
└──────┬──────────┘
       │
       ▼
┌─────────────────┐
│ CD: staging     │  ◄── automatic on main branch
│ ├─ deploy       │
│ ├─ integration  │
│ ├─ e2e tests    │
│ └─ load tests   │
└──────┬──────────┘
       │
       ▼
┌─────────────────┐
│ CD: production  │  ◄── manual approval OR automatic with strong gates
│ ├─ canary 5%    │
│ ├─ wait+observe │
│ ├─ canary 50%   │
│ ├─ wait+observe │
│ └─ full rollout │
└──────┬──────────┘
       │
       ▼
┌─────────────────┐
│ CT: production  │  ◄── continuous, never stops
│ ├─ synthetics   │
│ ├─ SLO monitor  │
│ └─ chaos tests  │
└─────────────────┘
```

See `lab.md` for hands-on exercise.
