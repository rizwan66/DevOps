# Lab 8: The MeridianPay Consulting Engagement

This is a role-play exercise. You are the consultant. Read the brief, then work through each phase. The "right" answer isn't a single solution — it's whether you applied the frameworks from the README without skipping steps. Resist the temptation to jump straight to recommendations.

Block out 3–4 hours for this. Do it in writing, not in your head. The artifacts you produce (notes, the proposal, the postmortem) are the actual deliverable.

---

## The brief

You've been brought in for a 6-week engagement at **MeridianPay**, a mid-sized payments company (~600 engineers, ~80 services). You report to the VP of Platform Engineering, **Dana Okafor**. The engagement was sold on this premise:

> "Our CI/CD is a mess. Deploys take forever, things break in production constantly, and our engineers are frustrated. We want you to come in and fix our pipelines."

That's all you got from the sales team. Dana is expecting you Monday morning.

Some unstructured intel from the kickoff call:
- They use GitHub Actions and Argo CD on EKS
- There's a "Platform Team" of 12 people that owns shared CI infrastructure
- There's a strong rumor that the Platform Team and the largest product team (Payments Core, ~80 engineers) are openly hostile to each other
- The previous Director of Platform left 4 months ago; Dana is new (started 6 weeks ago, hired externally)
- A "Pipeline Modernization Initiative" was launched 8 months ago by the previous director and is still ongoing. Nobody on the kickoff call could clearly articulate its current status.

---

## Phase 1: Week One (don't change anything)

You start Monday. Your calendar is empty except for a 30-minute intro with Dana on Tuesday and a recurring "Platform Team standup" invite she forwarded you.

**Your task for Phase 1:** produce a Week-One Plan before you do anything else. Write it down. Specifically:

1. **Who do you want to talk to in week 1, and in what order?** List 8–12 people by role (you don't know names yet). Justify the ordering — who do you talk to first and why? Who do you deliberately *not* talk to in week 1, and why?

2. **What questions do you ask each archetype?** Write 5–7 questions for each of: Dana, a Platform Team senior engineer, a Platform Team lead, a Payments Core senior engineer, a Payments Core engineering manager, an SRE, a junior engineer who joined in the last 6 months. Don't reuse questions across roles — tailor them.

3. **What do you read?** List the artifacts you ask for in week 1. Be specific (not "documentation" but "the runbook for the most recent SEV-1 incident"). Aim for 8–10 items.

4. **What metrics do you ask for, and from whom?** List 5–8 metrics. For each, name the likely owner of that data.

5. **What are you explicitly NOT doing in week 1?** Write down at least 5 things you're tempted to do but won't. (e.g., "I'm not going to open the Pipeline Modernization Jira board and start commenting on tickets.")

**Self-check:** Did you write any sentence that starts with "I think they should..."? If yes, delete it and start over. Week 1 is observation, not prescription.

---

## Phase 2: The Week-Two Bombshell

End of week 1, you've done your interviews. Here's the picture that emerged (this is given to you — you don't have to "discover" it):

- The **Pipeline Modernization Initiative** was supposed to migrate everyone from a legacy Jenkins setup to GitHub Actions. After 8 months: ~30 of 80 services migrated. The remaining 50 are stuck because Payments Core refuses to migrate — they have a custom Jenkins shared library that does things GitHub Actions can't easily do (or so they claim).
- The Platform Team built a set of "golden path" reusable workflows. Payments Core engineers complain these workflows are "too restrictive" and "don't fit their use case." They've forked them and maintain a divergent version.
- Argo CD is set up but only ~40% of services use it for deploys. The rest deploy via a homegrown bash script that SSHes into nodes. Yes, really.
- DORA metrics, roughly estimated:
  - Deploy frequency: weekly for ~20% of services, monthly for the rest
  - Lead time: 3–5 days median, 2–3 weeks for Payments Core
  - Change failure rate: ~25% (industry "elite" is <5%)
  - MTTR: ~6 hours median, but with huge variance — some incidents take days
- There's no shared observability stack. Some teams use Datadog, some use a self-hosted Prometheus, Payments Core has their own Grafana that nobody else can access.
- Dana wants you to "make a recommendation" by end of week 2.

**Your task for Phase 2:**

1. **Run the diagnostic frameworks from the README on what you've learned.** Specifically: write out the Five Whys for at least *two* of these problems: (a) why has the migration stalled? (b) why is change failure rate at 25%? (c) why are there parallel tool stacks? Don't accept the surface answer — push to the systemic cause.

2. **Apply Conway's Law analysis.** Sketch (in prose or ASCII) the org structure as you understand it. Then sketch the system architecture. Where do they mirror each other? Where is friction caused by the mismatch?

3. **Identify what's a *technical* problem and what's a *political/organizational* problem.** Be honest. The split is rarely 50/50.

4. **Write Dana's one-page proposal.** Use the template from the README. The 6-week engagement won't be enough to fix everything — the proposal must scope honestly. Include: problem statement, proposed approach (sequenced, not all at once), what's explicitly out of scope, success metrics with baselines and targets, what you need from Dana (decisions, air cover, headcount), risks, your role vs. internal team's role.

**Self-check:** Did you propose to "consolidate everyone onto GitHub Actions and Argo CD by end of engagement"? If yes, you've fallen into the consultant-hero trap. The migration has been stalled for 8 months for *reasons*, and you don't fully understand those reasons yet. A 6-week engagement cannot force a reluctant 80-person team to migrate. Rewrite.

---

## Phase 3: The pushback

You present your proposal to Dana. She likes it. She forwards it to her boss, the CTO, **Marcus Reed**. Marcus calls a meeting with you, Dana, and the head of Payments Core, **Priya Shankar**. Halfway through, Marcus says:

> "I appreciate the thoughtful proposal, but I need this faster. I've committed to the board that we'll have a 'unified deployment platform' by end of Q2 — that's 10 weeks from now. Forget the 6-week scoping; I need you to lead the execution. Get Payments Core onto the golden path. I'll back you up if Priya pushes back."

Priya, sitting two seats away, says nothing. Her face is unreadable.

**Your task for Phase 3:**

1. **What are the three pushback options from the README, applied to this situation?** Write out:
   - **Comply:** what does compliance look like? What are the likely outcomes (good and bad)?
   - **Negotiate:** what's the negotiated position? What do you give, what do you ask for in return? How do you frame it to Marcus without seeming to undermine him in front of Priya?
   - **Refuse:** what does refusal look like? What are the consequences? Is this a hill worth dying on?

2. **What do you actually say in the meeting?** Write the script. Real words. 3–5 sentences max — don't filibuster. Account for the fact that Priya is in the room and you've been put on her opposite side without your consent.

3. **What do you do in the next 48 hours regardless of the outcome?** Specifically: who do you talk to, what do you write down, what do you not commit to in writing yet?

**Self-check:** Did you say yes to Marcus to keep him happy? You will own the failure when Q2 ends and Payments Core hasn't migrated. Did you say no flatly? You may have ended the engagement prematurely without exploring the negotiated middle. The art is in the middle.

---

## Phase 4: The incident

Three weeks into execution, an incident happens. A deploy of `payments-core-api` (one of the Payments Core services that *did* successfully migrate to the golden path two weeks ago) caused a 47-minute partial outage during business hours. Customer-facing checkout was degraded; estimated revenue impact $1.2M.

The post-incident slack thread is heated. A Payments Core engineer, **Tom Reilly**, writes:

> "This is exactly what we warned about. The 'golden path' migration broke our deploy. We had a perfectly working Jenkins pipeline. Now we have an outage. This whole initiative is a disaster."

A Platform Team engineer, **Aisha Patel**, replies:

> "The golden path worked fine. The bug was in your service config — you set the readiness probe wrong. That's on you, not the pipeline."

Dana asks you to run the postmortem.

**Your task for Phase 4:**

1. **Before the postmortem meeting,** what do you do to prepare? List 6–8 actions. (Hint: read the actual incident data, talk to Tom and Aisha separately, look at the deploy diff, check the runbook.)

2. **Run the blameless postmortem** using the template from the README. Fill it in with plausible details — you're allowed to invent specifics as long as they're consistent with what you know. The contributing factors section is the most important: there are at least 4 contributing factors here, and "Tom set the readiness probe wrong" is only one of them. Find the others.

3. **Write the action items.** Each must have an owner and a due date. Do NOT assign all action items to the Platform Team. Do NOT assign all action items to Payments Core. The point of blameless postmortems is to fix systemic issues, which means action items distributed across teams and possibly leadership.

4. **Politically:** how do you handle Tom's framing in the postmortem meeting? He's set up a narrative ("the migration caused this"). If you let it stand unchallenged, the migration stalls. If you dismiss it, you've broken the blameless principle and Payments Core never trusts you again. Write the 2–3 sentences you'd say to reframe.

**Self-check:** Did your action items include "Tom needs more training"? That's a blame-shaped action item wearing a polite mask. Did your action items include changes to the golden path itself? They probably should — the readiness probe was set wrong because the migration tooling didn't validate it. That's a systemic gap.

---

## Phase 5: Handoff

It's week 6. Engagement is ending (you and Dana negotiated extending Marcus's timeline; the migration is on track but not complete — Payments Core has migrated 6 of 12 services, and the team is now trained well enough to migrate the rest themselves over the next 2 months).

**Your task for Phase 5:**

1. **Write the handoff document** using the template from the README. Real artifact, real prose. Topics to cover at minimum:
   - The history of how things got to where they are (so the next person doesn't have to re-discover it)
   - What's done, what's in progress, what's not started
   - The political landscape: who's a champion, who's a skeptic, who's a blocker, and *why*
   - The decisions that are pending (and what would happen under different choices)
   - The traps — things that look obvious but have hidden dependencies
   - Who owns what now that you're leaving
   - What you would do if you were staying

2. **Identify the named successor** for each thing you owned during the engagement. If there isn't one for some piece, that's a red flag — flag it explicitly.

3. **Write your final email to the org.** Short. Not self-congratulatory. Names the people who did the actual work. Sets up the next chapter.

**Self-check:** Did you list yourself as a "point of contact for questions" after you leave? Don't. The engagement is ending. Your absence is the test of whether you actually transferred knowledge.

---

## Reflection

After completing all five phases, write a one-page reflection answering these questions honestly:

1. Where in the role-play did you want to skip a step? (Most likely: phase 1 — you wanted to start solving before listening.)
2. Where did you find yourself wanting to pick a side between Platform and Payments Core?
3. If you actually did this engagement and Marcus had said what he said in Phase 3, what would your real reaction have been? Anger? Capitulation? Avoidance? Notice it. The frameworks help, but only if you can stay calm enough to apply them under pressure.
4. What's the single most important thing you learned that the README didn't make obvious?

---

## Beyond this lab

Real consulting engagements are messier than this. People don't fit archetypes. The political situation evolves daily. New problems surface. Sometimes the engagement gets cut short, or extended past usefulness, or the person who hired you leaves mid-engagement.

The frameworks in this module aren't a script. They're scaffolding for thinking clearly when the situation is unclear and the pressure is high. The only way to internalize them is to use them, get them wrong, and refine.

If you want to push further: do this exercise again in 3 months without re-reading the README first. See what you remember. The gaps tell you what to study.
