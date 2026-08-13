# Behavioral & Leadership Questions (Senior IC Bar)

At the senior level, behavioral rounds probe for **scope of ownership, technical judgment under ambiguity, and influence without authority** — not just "tell me about a time." Prepare 5-8 core stories (STAR: Situation, Task, Action, Result) that you can reshape to answer many of these, rather than memorizing 30 separate stories. Infra-specific loops especially probe incident response and cross-team technical influence.

## Story bank to prepare (aim for range across these)
Your prepared stories live in `../notes/star-stories.md` (11 STAR stories from EBS Direct / EBS Data Lifecycle Manager work, plus 3 project highlights). Cross-links to specific questions are below; gaps are marked *(no story yet — needs prep)*.

- A production incident you led or were deeply involved in resolving.
- A time you disagreed with a technical decision and had to push back (ideally where you were right, and one where you were wrong/changed your mind).
- A project where you influenced other teams without formal authority over them.
- A time you made a call to cut scope or ship something imperfect under a deadline, and how you managed the risk.
- A time you found and fixed a systemic problem (not a one-off bug) — process, tooling, or architecture.
- A mentoring/leveling-up-others story.
- A time you were wrong about a technical bet and had to unwind it.
- A time you had to make a decision with incomplete information.

Each question below is tagged with the best-fit story from `../notes/star-stories.md` where one exists (⭐ *your story:* `#story-id`). Untagged questions are gaps — draft or adapt a story for those before the interview rather than improvising cold.

## Incident / operational excellence
- Tell me about the most significant production incident you've been involved in. What was your role, what did you do, what was the outcome?
  - ⭐ *your story:* `#archival-feature` (reframe around the monitoring/alerts/runbooks you set up, even though it's design-led rather than a live-fire incident — be upfront that it's not a true incident story if asked directly).
- Describe a time you paged yourself or others in error, or a time an alert was too noisy — what did you do about it?
  - *(no story yet — needs prep; this is a real gap since none of your prepared stories are on-call/paging-specific)*
- Tell me about a postmortem you wrote or participated in. What was the actual root cause, and what changed as a result?
  - ⭐ *your story:* `#pipeline-fix-vs-bypass` (root cause was a dependency bug in another team's service; the change was adding canary checks and alarms).
- Describe a time you had to make a risky operational call (rollback, failover, take a system down) under time pressure.
  - ⭐ *your story:* `#pipeline-fix-vs-bypass` (fix-vs-bypass decision under deadline pressure on a region launch).
- How do you decide what should page someone at 3am vs wait until morning?
  - *(no story yet — needs prep; answer from principles, e.g., customer-facing/data-loss-risk pages now, everything else waits)*
- Tell me about a time you prevented an incident before it happened.
  - ⭐ *your story:* `#adc-region-mistake` (added the CI/CD validation check that catches outdated/missing configs before deployment) or `#pipeline-fix-vs-bypass` (added canary checks/alarms).

## Technical judgment & tradeoffs
- Tell me about a time you chose a simpler solution over a more "correct" one, and how you justified that call.
  - ⭐ *your story:* `#cicd-bug-outside-scope` (temporary safeguard in test setup instead of waiting on the other team's full fix).
- Describe a technical decision you made that you'd make differently today. What did you learn?
  - ⭐ *your story:* `#adc-region-mistake` (hardcoding region dependencies — your strongest genuine-mistake story).
- Tell me about a time you had to balance reliability/correctness against shipping speed.
  - ⭐ *your story:* `#pipeline-fix-vs-bypass`.
- Describe the most complex distributed system you've designed or operated. What made it hard, and what would you change?
  - ⭐ *your story:* `#archival-feature` (async workflows across API, Lambda, DynamoDB, scheduler, and worker).
- Tell me about a time you pushed back on a deadline or scope because of a technical/reliability concern.
  - ⭐ *your story:* `#pipeline-fix-vs-bypass`.
- How do you approach evaluating a build-vs-buy decision for infrastructure?
  - *(no story yet — needs prep; none of your prepared stories are build-vs-buy specific)*

## Influence & cross-team collaboration
- Tell me about a time you had to convince another team to change how they were doing something, without having authority over them.
  - ⭐ *your story:* `#dlm-console-ux` (convinced the UX team to approve a custom component) or `#cross-team-unblock`.
- Describe a time you disagreed with your manager or a senior engineer's technical direction. What did you do?
  - ⭐ *your story:* `#dlm-console-ux` (closest fit — disagreement was with the UX team rather than a manager/senior engineer; flag that distinction if asked precisely).
- Tell me about a project that required deep coordination across multiple teams. What was your role in making it succeed?
  - ⭐ *your story:* `#region-standardization` or `#recycle-bin-backend-comms`.
- Describe a time you had to say no to a request from a stakeholder (product, another eng team) and how you handled it.
  - ⭐ *your story:* `#dlm-console-ux` (pushed back on the UX team's initial fixed-workflow suggestion).
- Tell me about a time you had to explain a technical tradeoff to a non-technical stakeholder.
  - ⭐ *your story:* `#dlm-console-ux` (explained UX impact of the fixed-workflow component to the UX team).

## Ownership & scope
- Tell me about the largest-scope project you've owned end to end. How did you break it down?
  - ⭐ *your story:* `#archival-feature` (end-to-end owner for a five-person team).
- Describe a time you identified a problem nobody had asked you to solve, and drove the fix.
  - ⭐ *your story:* `#region-standardization` (self-initiated) or `#cicd-bug-outside-scope`.
- Tell me about a time you inherited a system in bad shape. What did you do?
  - *(no story yet — needs prep; `#adc-region-mistake` is close but it's a system you built, not inherited — draft a distinct story if you have one, e.g., anything pre-existing you took over)*
- How do you decide what NOT to work on when everything feels important?
  - *(no story yet — needs prep)*
- Tell me about technical debt you deliberately took on, and how you managed paying it down (or not).
  - ⭐ *your story:* `#cicd-bug-outside-scope` (the temporary safeguard was deliberate debt — mention whether/how it was later replaced by the other team's real fix).

## Growth & self-awareness
- Tell me about the biggest mistake you've made in your career. What did you learn?
  - ⭐ *your story:* `#adc-region-mistake` — lead with this one; it's your clearest genuine-failure story with a concrete fix and metric (~30% faster region setup).
- Describe feedback you received that was hard to hear. What did you do with it?
  - ⭐ *your story:* `#manager-feedback-bigger-picture` or `#recycle-bin-backend-comms`.
- Tell me about a time you had to learn a new domain/technology quickly to be effective.
  - *(no strong story yet — `#dlm-console-ux` touches new UI-pattern territory but isn't really "new domain"; consider drafting one, e.g., ramping on AWS Lambda/DynamoDB for the archival feature)*
- How has your approach to system design changed over your career?
  - ⭐ *your story:* `#manager-feedback-bigger-picture`.
- Tell me about a time you mentored someone through a hard technical problem, without just giving them the answer.
  - ⭐ *your story:* `#archival-feature` (onboarded engineers with mock APIs and staging tests — thin fit; strengthen with a specific mentoring anecdote if you have one).

## Infra-specific flavor questions
- How do you think about the tradeoff between building a general platform vs a point solution for one team's need?
  - ⭐ *your story:* `#region-standardization` (dynamic config module + reusable CI/CD templates, later adopted as the team standard).
- Tell me about a time you had to migrate a system/service with zero (or near-zero) downtime. Walk me through the plan.
  - ⭐ *your story:* the DLM Console Migration (GWT → React) project highlight in `../notes/star-stories.md` — expand it into a full STAR before the interview, it's currently only a one-line highlight.
- Describe how you've approached deprecating a widely-used internal system or API.
  - *(no story yet — needs prep; the GWT→React migration is adjacent but framed as a modernization, not a deprecation — consider reframing if pushed)*
- Tell me about a time capacity planning went wrong (under- or over-provisioned) and what you did.
  - *(no story yet — needs prep)*
- How do you approach on-call load and sustainability for a team you're on or leading technically?
  - *(no story yet — needs prep)*

## Prep checklist
- For every story: know the concrete numbers (scale, time saved, incident duration, blast radius) — vague stories read as junior.
- Practice the "what would you do differently" follow-up for every story — interviewers almost always ask it, and "nothing" is a red flag at senior level.
- Have at least one story where the result was a genuine failure or you were wrong — refusing to show any failure reads as low self-awareness.
- Tie stories back to principles/impact, not just mechanics ("I did X, Y, Z" is necessary but not sufficient — say *why* you made each call).
