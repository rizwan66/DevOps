# Module 8 — Consulting & SW integration playbook

> *"Consulting and collaborating with running projects SW integration teams, to optimize their CI/CD services and pipelines."*

This module has no YAML in it. That's deliberate. It's the part of the job that determines whether the YAML you write actually gets used.

## What "consulting" looks like inside a company

You're an internal consultant when you're embedded with a project team for a few weeks to months, helping them improve something specific. The team isn't your team. You don't have authority over them. You leave when you're done.

This is a fundamentally different role from "I own this thing forever". It rewards a different set of skills:

- Listening before talking
- Showing rather than telling
- Leaving the team better off than you found them, *and able to maintain the change*
- Knowing when to push back on a request you've been given
- Knowing when to walk away

## The first week: don't change anything

Resist the urge to immediately propose changes. The first week is for **understanding the existing system**, the people, and the implicit constraints. If you start proposing changes on day one, you'll be the smart person who came in with answers before understanding the questions, and the team will quietly ignore you.

What to do in the first week:

### Day 1–2: shadow people doing real work
- Watch the lead developer make a code change end to end.
- Sit with whoever runs deployments while they run one.
- Watch the on-call engineer respond to an alert, even a fake test alert.

Take notes. Don't suggest anything yet.

### Day 3–4: read everything
- Their CI configs. All of them.
- Their runbooks (or lack thereof).
- The last six months of incident postmortems.
- The Slack channels where engineering happens. Read backwards.

You're looking for: pain points people have stopped mentioning because they've given up. Recurring questions. "We tried X but..." stories.

### Day 5: the listening interviews
Schedule 30-minute one-on-ones with 4–6 people across roles. Ask three questions:

1. "If you could change one thing about how we ship software, what would it be?"
2. "What's the most frustrating part of your week?"
3. "What's something that works well that I shouldn't break?"

The third question is the most important. It surfaces the things you might accidentally damage. It also signals "I'm here to help, not to change everything for the sake of changing things."

### End of week 1: write the brief

Send the team (or the manager who hired you) a short document:

```
Subject: Week 1 observations from [your name]

What I've heard:
- [3–5 themes from the interviews]

What I think the top opportunities are:
- [3 bullets, ranked]

What I'm proposing for week 2:
- [Concrete, scoped first action]

What I'm explicitly NOT proposing yet:
- [Things you've heard people want but aren't ready for]

Open questions:
- [Things you genuinely don't know yet]
```

This document does several things at once. It demonstrates you listened. It signals priorities. It explicitly *defers* tempting work that's not yet justified. It invites correction.

## Diagnostic frameworks for SW integration teams

When you're consulting on a CI/CD pipeline, here's a checklist of common pathologies and what to look for. Tick the ones that apply.

### Diagnostic 1 — Pipeline duration & feedback loop
- [ ] How long does the pipeline take from `git push` to "all green"?
- [ ] What % of pipeline runs result in a failure that's not the developer's fault (flaky tests, infra issues)?
- [ ] If a pipeline fails, how long until the responsible developer notices? (Check Slack/email integration latency.)

**Threshold**: > 15 min pipeline = developers context-switch and forget. > 5% flake rate = developers ignore failures and rerun without thinking.

### Diagnostic 2 — Deploy mechanics
- [ ] Who can deploy to production? How many people? Are deploys automated or manual?
- [ ] How do you know a deploy succeeded?
- [ ] How do you roll back? Has anyone done it in the last 6 months?

**Threshold**: < 3 people who can deploy = bus factor problem. > 10 = lack of standardisation. Rollback never tested = it doesn't work.

### Diagnostic 3 — Test pyramid
- [ ] How many unit tests? Integration tests? E2E tests? In what proportion?
- [ ] How long does each tier take to run?
- [ ] When did you last delete a test that wasn't earning its keep?

**Threshold**: e2e-heavy pyramids are slow and flaky. The shape should be unit > integration > e2e by ratio.

### Diagnostic 4 — Configuration management
- [ ] Where are environment-specific values stored? In code? In secret managers? In config maps?
- [ ] What happens to a config value when it's outdated? Does anyone clean up?
- [ ] How does a developer test a config change?

**Threshold**: config in three different systems = drift incoming. No cleanup = fossilised cruft.

### Diagnostic 5 — Knowledge concentration
- [ ] How many "ask Marta" steps are there in the deploy process?
- [ ] What happens if the lead engineer is on holiday?
- [ ] Are runbooks current? When were they last edited?

**Threshold**: any "ask X" step is a single point of failure waiting for X to leave the company.

### Diagnostic 6 — Observability
- [ ] Can you tell, in real time, that a deploy made things worse?
- [ ] How long after a customer reports an issue do you confirm it?
- [ ] Are alerts respected, or is the alerts channel muted?

**Threshold**: muted alerts = the team has given up on the alerting system. That's worse than no alerts.

## Optimising someone else's pipeline: a sequenced approach

When you're brought in specifically to "optimise their CI/CD pipelines", here's the order of operations that tends to produce results without breaking things.

### Step 1: Measure (week 1)
Add observability *before* changing anything. You can't claim improvement without before/after numbers.

- Instrument pipeline durations (most CI systems expose this).
- Track flake rate (failures that pass on retry without code change).
- Track lead time (commit to deploy).
- Get baseline DORA metrics.

### Step 2: Quick wins (week 2–3)
Pick changes that are obvious wins, low risk, and can be evaluated quickly.

- Add caching where there isn't any.
- Parallelise sequential jobs that don't actually depend on each other.
- Remove dead steps (the ones nobody knows the purpose of). Run for two weeks, see if anyone complains.
- Fix the top 3 flaky tests.

These changes will speed up the pipeline 2–3× without changing how anyone works. Now you have credibility for harder things.

### Step 3: Standardise (week 4–6)
Now you can suggest structural changes.

- Convert per-repo CI YAML to a shared reusable workflow.
- Standardise on a single base image, a single Python version, a single test runner.
- Centralise secret management.

Expect pushback here. Each team had reasons for their snowflake setup; you have to address those reasons, not just override them.

### Step 4: Fundamental shifts (month 2–3)
Only now is it worth proposing big changes:

- Push-to-pull (GitOps).
- New language/runtime upgrades.
- Cross-team platform.

These are 6-month projects, not 6-week projects. Be honest about that.

## Pushback and saying no

You'll be asked to do things that are wrong. Examples:

- "Skip the tests for this hotfix; we'll add them later."
- "Just hard-code the prod credentials in CI for now."
- "We need to ship by Friday — disable the security scan."

Three options when you're asked something like this:

### Option A — Comply, document the risk
Sometimes the business need really does outweigh the technical concern. Comply, but write down the risk you're accepting and email it to the requester. The email creates a paper trail. If it later goes wrong, the conversation is "we knew this was a risk, here's what we said about it" rather than "you told us to do this and now you're blaming us".

### Option B — Negotiate a smaller version of the request
"I can't disable the security scan, but I can give you a one-day exception with auto-revert if it's important to ship today." You bend without breaking.

### Option C — Refuse
Reserve this for the small number of cases where the request is genuinely dangerous and there's no acceptable middle ground. Be specific about *why*: "I won't hard-code prod credentials because if the repo is ever made public, we have a breach. Here's an alternative that solves your actual need."

The order matters. Default to compliance with documented risk; reserve refusal for genuine disasters. Senior consultants who refuse a lot get a reputation for being unhelpful and stop being invited to important conversations. Senior consultants who comply with everything get a reputation for not having a backbone. Pick your battles.

## Knowledge transfer: the part everyone skips

The whole point of consulting is that you eventually leave. If when you leave, the team can't maintain what you built, you've done bad work — even if the work itself is technically excellent.

Things to do in your last weeks:

- **Pair, don't solo.** Whatever you build, build it with someone from the team watching. Ideally that person commits the code with you reviewing.
- **Write the runbook.** Not just "how does this work" but "what to do when it breaks".
- **Explicit handoff session.** Walk through every component you built, with the people who'll own it. Get them to demo it back to you. Then take questions.
- **Defined exit.** Don't drift away. Have a date. Have an explicit "I'm leaving on Friday; here's the handoff doc; here's where to ping me for one month afterwards".

## Communication patterns

A few specific patterns that work well:

### "Here's what I'm seeing, here's what I'm thinking, here's what I'd like to do"
Three-part structure for any proposal. Observations → analysis → action. Forces you to ground recommendations in reality and makes it easy for someone to push back at any of the three stages.

### "Stupid question, but..."
Useful framing when you genuinely don't understand something. It gives the other person space to explain without feeling tested. It also signals that you're not pretending to know things you don't.

### "What problem are we trying to solve?"
The most useful question in any meeting. People propose solutions before they've agreed on the problem. Bringing the discussion back to the problem clarifies whether the solution is even relevant.

### Don't say "easy"
"Oh that's easy, I'll knock it out tomorrow." Three weeks later, you're still working on it, the team has lost confidence, and you've taught them not to trust your estimates.

### Postmortems: blameless and concrete
When something fails, the writeup should focus on the system that allowed the failure, not the individual who triggered it. "Why did the deployment process let this through?" not "Why did Bob push the wrong commit?".

## Templates

A few short templates you can adapt.

### The one-page proposal

```
PROPOSAL: [single descriptive title]
AUTHOR: [you]
DATE: [today]

PROBLEM
- One paragraph. Quantified if possible.

PROPOSAL
- One paragraph describing the change.

NON-GOALS
- Things this proposal explicitly does NOT change.

SUCCESS METRICS
- 1–3 metrics that will tell us if it worked. Specific values.

ROLLBACK PLAN
- If this turns out to be a bad idea, what do we do?

TIMELINE
- Rough weeks/days estimate.
```

### The blameless postmortem

```
POSTMORTEM: [what happened, briefly]

IMPACT
- Duration, scope (users affected, services affected, $ if applicable).

TIMELINE
- Bullet list with timestamps. Just facts.

ROOT CAUSE
- The system reason this happened. NOT the person.

WHAT WENT WELL
- Things that worked as intended.

WHAT WENT POORLY
- Things that didn't work.

ACTION ITEMS
- Specific, owned, dated items. Each must address something concrete.
```

### The handoff doc

```
HANDOFF: [system/component name]

WHAT IT IS
- One paragraph.

WHO OWNS IT NOW
- Name(s).

WHERE THINGS LIVE
- Code: [link]
- Runbook: [link]
- Dashboard: [link]
- Alerts: [link]

HOW TO CHANGE IT
- Step-by-step for a normal change.

HOW TO ROLLBACK
- Step-by-step for the bad day.

WHO TO ASK
- For 1 month: [you]
- After that: [team chat / specific people]

KNOWN ISSUES / TECH DEBT
- Things that are imperfect but not worth fixing now.
```

## See `lab.md` for the role-play exercise.
