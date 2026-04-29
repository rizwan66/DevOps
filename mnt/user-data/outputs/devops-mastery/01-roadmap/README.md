# Module 1 — DevOps initiatives & end-to-end workflows

> *"Driving DevOps initiatives and facilitating end-to-end project workflows."*

This module is about the **glue**. The glue is the part of DevOps that doesn't show up in a YAML file but determines whether your YAML files actually deliver value. Skip this module and you'll write technically correct pipelines that nobody uses.

## What "end-to-end workflow" actually means

A workflow is end-to-end when a single change can travel from a developer's keyboard to a user's screen without anyone manually copying anything between systems. Most "DevOps" setups in the wild are *not* end-to-end. They have automated pieces with manual handoffs in between — someone has to email a build artifact, click a button in Jenkins, ping the SRE on Slack, or open a Jira ticket for the deployment.

Each manual handoff is a place where:

- Knowledge gets lost (only Sarah knows how to do step 4)
- Errors get introduced (someone deploys the wrong tag)
- Delivery slows down (Sarah is on holiday)
- Auditability disappears (no record of who approved what)

Your job as the person "driving DevOps initiatives" is to find those handoffs and replace them with code, configuration, or — when automation isn't appropriate — explicit policy.

## The DORA metrics: how you know you're succeeding

DORA (DevOps Research and Assessment, originally from Google) defines four metrics that correlate strongly with business outcomes. Memorise these, because they're the language senior leadership uses.

| Metric | What it measures | Elite | Low |
|--------|------------------|-------|-----|
| **Deployment frequency** | How often you ship to production | Multiple per day | Less than monthly |
| **Lead time for changes** | Commit to production | Less than 1 day | More than 6 months |
| **Change failure rate** | % of deploys that cause an incident | 0–15% | 46–60% |
| **Mean time to recovery** | How long from incident detected to resolved | Less than 1 hour | More than 1 week |

Two important nuances:

1. **The metrics are coupled.** If you optimise deployment frequency by skipping tests, your change failure rate will skyrocket. If you optimise change failure rate by deploying once a quarter with twelve approvers, your lead time will be measured in weeks. Good DevOps moves all four in the right direction simultaneously.
2. **You measure them per-team, not per-company.** Aggregating across teams hides the teams that are struggling.

## A second framework: the Westrum typology

Westrum classifies organisational cultures into three types, and DORA research shows that culture predicts performance more reliably than any specific tool. Recognise which culture you're consulting into before you propose changes:

- **Pathological** (power-oriented): Information is hoarded. Failure is punished. New ideas are crushed. → **Don't propose GitOps here yet. Propose a blameless postmortem first.**
- **Bureaucratic** (rule-oriented): Information is ignored. Failure leads to justice. New ideas create problems. → **Standardise existing processes; don't introduce new tools.**
- **Generative** (performance-oriented): Information is actively sought. Failure leads to enquiry. New ideas are welcomed. → **This is where ambitious DevOps initiatives actually land.**

## Driving an initiative: a concrete framework

When someone says "drive a DevOps initiative", they mean: identify a problem, propose a solution, get buy-in, deliver it, and prove it worked. Here's a structure that works:

### 1. Find the bottleneck (don't propose solutions yet)

Sit with each team for an afternoon and watch them ship something. Don't talk. Just take notes. You're looking for:

- Steps that take longer than expected
- Places where someone says "I'll just SSH into the box quickly"
- Recurring questions in Slack like "anyone know how to redeploy staging?"
- Manual steps people don't even notice anymore because they're so used to them

Quantify what you find: *"Team X spends 4 hours per release on manual verification, and releases happen weekly. That's 200 engineer-hours per year."*

### 2. Pick the right battle

Not every bottleneck is worth fixing. Use a 2×2 of **impact × effort**:

```
              Low effort      High effort
High impact   ▶ DO FIRST      ▶ Plan carefully
Low impact    ▶ Quick win     ▶ Don't bother
```

Resist the urge to tackle the hardest problem first to prove your worth. Pick a quick win to build credibility, then use that credibility to tackle the hard problem.

### 3. Write a one-page proposal

Not a 40-slide deck. One page. It should answer:

- What problem are we solving? (with metrics)
- What are we proposing to change?
- What are we *not* changing? (this is reassuring to skeptics)
- How will we know if it worked? (metrics again)
- What's the rollback plan if it doesn't?

### 4. Build it iteratively, with one team first

Don't roll out to all twelve teams on day one. Pick the most enthusiastic team, deliver value to them, and let them become advocates. Other teams will ask to be next. This is *much* easier than top-down rollouts.

### 5. Measure and report honestly

After the rollout, report the actual numbers. If they're worse than before, say so. Credibility compounds; one honest "this didn't work as expected, here's what we learned" is worth more than ten "the project was a great success" reports.

## End-to-end workflow design: the seven stages

A complete software delivery workflow has roughly these stages. Map your current state against this list to find the gaps.

| Stage | What happens | Who owns it | Tools |
|-------|--------------|-------------|-------|
| 1. Plan | Work is identified, prioritised, broken down | PM / Tech lead | Jira, Linear, GitHub Projects |
| 2. Code | Developer writes the change | Developer | IDE, Git |
| 3. Build | Code is compiled, packaged into artifacts | CI system | GitHub Actions, Jenkins, GitLab CI |
| 4. Test | Automated tests run; quality gates pass | CI system | unit/integration/e2e test runners |
| 5. Release | Artifact is promoted toward production | CI/CD | GitOps tools, Argo, Flux, Spinnaker |
| 6. Deploy | Artifact runs in production | CD / GitOps | Kubernetes, ECS, Nomad |
| 7. Operate | Service is monitored, incidents are handled | SRE / on-call | Prometheus, Grafana, PagerDuty |

**The "DevOps engineer" role typically spans stages 3–7.** Stage 1 is product, stage 2 is the developer. But you'll be more effective if you understand all seven, because the bottleneck is often *between* stages, not within them.

## Anti-patterns to recognise

Things that look like DevOps but aren't:

- **"DevOps team" as a separate silo.** If there's a team called "DevOps" that other teams throw tickets at, you've recreated the old Ops team with new branding. The point of DevOps is to remove that wall.
- **Tool-driven transformations.** "We're going to do DevOps by adopting Kubernetes." Kubernetes is a tool. DevOps is a way of working. Adopting tools without changing how you work is how you end up with a worse version of the old system.
- **CI without CD.** Lots of teams have automated builds and tests but still deploy by hand. They've automated the easy 30% of the problem.
- **CD without monitoring.** Shipping to production fast without observability is a way to ship bugs to production fast.
- **Pipelines as cargo cult.** Every project has the same 14-step pipeline copy-pasted from the last project, with steps no one remembers the purpose of. Periodically delete steps and see if anyone notices.

## See the lab

Open `lab.md` in this folder for the hands-on exercise: auditing a fictional team's delivery process and writing the one-page proposal.
