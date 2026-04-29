# Module 1 — Lab

## The scenario

You've been brought in to consult with **Team Orion**, a 12-person team building a payments microservice. Here's what you observe over a week of shadowing them:

- They use GitHub for source control. Pull requests typically sit open for 2–4 days waiting for review.
- They have a Jenkins server (run by a separate Infra team) that builds Docker images on every push to `main`. The pipeline takes 38 minutes.
- After Jenkins builds, the lead engineer Marta SSHes into the staging server and runs `docker pull && docker restart payments-svc`. She does this maybe twice a week.
- Production deployments happen every other Friday at 4pm. The release process is a 23-step Confluence page that hasn't been updated in seven months. Three of the steps say "ask Marta".
- They have monitoring: a Grafana dashboard that nobody looks at, and Slack alerts that everyone has muted because there are too many false positives.
- The last three production incidents were all caused by configuration that worked in staging but not production. Average time to recover: 4 hours.
- When you ask "how do you know if a deployment was successful?", you get four different answers from four different engineers.

## Your tasks

### Task A — Map the current workflow

Draw a diagram (ASCII or whatever tool you like) showing the seven stages from the README and what Team Orion's current state looks like at each stage. Mark the manual handoffs in red.

### Task B — Estimate the DORA metrics

Based on what you've observed, estimate Team Orion's four DORA metrics. You don't have hard data, so make reasonable assumptions and state them. Place them in the elite/high/medium/low brackets.

### Task C — Identify the top three bottlenecks

Pick the three biggest problems. For each one, write:

- The problem in one sentence
- Roughly how much engineering time it costs the team per month
- An estimate of effort to fix (small / medium / large)
- An estimate of impact (small / medium / large)

Plot them on the impact/effort 2×2 from the README.

### Task D — Write the one-page proposal

Pick **one** of the three bottlenecks (the one with the best impact/effort ratio) and write the one-page proposal as described in the README. Five sections: problem, proposal, non-goals, success metrics, rollback plan. No more than 500 words total.

### Task E — Anticipate objections

For your proposal, list the three most likely objections you'll hear and write a one-sentence response to each. Examples of common objections:

- "We don't have time to do this right now."
- "The Infra team owns Jenkins, not us."
- "We tried this two years ago and it didn't work."

## Reflection questions

After completing the tasks, answer these for yourself (not for anyone else — the point is to see if you actually internalised the framework):

1. Did you propose a tool, or did you propose a change in how the team works? If only a tool, you fell into the trap. Try again.
2. Is your proposal something the team can do without permission from anyone? If it requires three approvals from three VPs, it's too ambitious for a first initiative.
3. If your proposal works perfectly, which of the four DORA metrics will move? By how much? If you can't answer this, you don't have a clear enough success criterion.
4. What's the one piece of evidence that would convince you your proposal *won't* work? (If you can't think of any, you're not being honest with yourself about the risks.)

## What good looks like

A senior DevOps engineer would do tasks A–E in roughly 90 minutes, not 90 days. The point is not to produce a perfect plan; the point is to develop the **reflex** of breaking down a vague "things are slow" complaint into a quantified, prioritised, low-risk first move.

If you find yourself wanting to propose "let's adopt FluxCD and Terraform and Prometheus and rewrite all the CI pipelines", stop. That's a two-year programme, not a first initiative. Pick the one thing that's costing the team the most time *right now*, with the least disruption, and start there.
