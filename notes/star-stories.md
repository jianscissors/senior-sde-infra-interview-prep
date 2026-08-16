# STAR Story Bank

Your prepared behavioral stories, cleaned up into consistent STAR format. Each story has a short ID (e.g. `#archival-feature`) used to cross-link it from `../interview-questions/03-behavioral-and-leadership-questions.md` — reshape/reweight these per question rather than memorizing separate answers for every prompt.

---

## #archival-feature — Archival Feature for Data Lifecycle Manager (Amazon EBS)

**Situation:** Led the archival feature at Amazon EBS, letting customers move long-lived snapshots to cold storage. Touched multiple backend systems and introduced new async workflows.

**Task:** End-to-end design and execution owner for a five-person team — define retries/fallbacks, coordinate the team's efforts.

**Action:**
- Introduced idempotency tokens and state-machine–style transitions for failure handling.
- Documented failure points and tradeoffs; led design reviews.
- Helped engineers onboard with mock APIs and staging tests.
- Set up monitoring, alerts, and runbooks.

**Result:** Launched ahead of schedule; customer adoption growing; system reliability >99.9%; the failure-handling patterns were later reused on similar features.

**Good for:** ownership/scope, technical judgment & tradeoffs, ambiguous requirements, incident/operational excellence (monitoring & runbooks), largest-scope project owned.

---

## #region-standardization — Region Build Standardization Project (EBS Direct)

**Situation:** During EBS Direct's expansion, every region had different ADC configurations and service dependencies, making deployments error-prone and slow.

**Task:** Proactively proposed standardizing multi-region deployments for repeatability and scale — self-initiated, not assigned.

**Action:**
- Designed a dynamic configuration module replacing hard-coded region info.
- Created standardized CI/CD pipeline templates.
- Shared the design and guided other engineers on adopting the module.
- Added validation checks for configuration accuracy.

**Result:** Region deployment became fully repeatable and reliable; reduced debugging/troubleshooting time; adopted as the team standard for onboarding new regions.

**Good for:** identified a problem nobody asked you to solve, influence without authority, technical judgment, largest-scope project, "how do you decide what to work on."

---

## #cross-team-unblock — Cross-Team Collaboration During Multi-Region Build

**Situation:** During the EBS Direct multi-region build, a deployment was repeatedly blocked by failures traced to another team's infrastructure pipeline.

**Task:** Unblock the region build despite the root cause sitting outside your own team.

**Action:**
- Took ownership of the unblock rather than waiting on the other team.
- Investigated the root cause and filed a detailed ticket.
- Worked directly with the partner team to identify and fix the underlying bug.

**Result:** Region builds completed smoothly and ahead of schedule, delivering faster, more reliable access for customers globally.

**Good for:** influence without authority, cross-team collaboration, ownership beyond your team's boundary.

---

## #cicd-bug-outside-scope — CI/CD Pipeline Bug Found Outside Core Scope (DLM)

**Situation:** While writing integration tests for the Pre/Post Hook feature in DLM, noticed the CI/CD pipeline intermittently failing, slowing deployments and blocking other engineers.

**Task:** Keep Pre/Post Hook tests running reliably without blocking the team, even though pipeline maintenance wasn't your core responsibility.

**Action:**
- Investigated the failures and traced the root cause to another team's service dependency.
- Opened a detailed ticket for the responsible team.
- Implemented temporary safeguards in the integration test setup as a stopgap.
- Documented the workaround and shared it with the team.

**Result:** Pipeline stabilized; integration tests ran reliably; development continued without delay; the Pre/Post Hook feature launched smoothly.

**Good for:** ownership beyond assigned scope, "found a problem nobody asked you to solve," pragmatic tradeoffs (temporary safeguard vs waiting for a full fix).

---

## #ambiguous-failure-handling — Handling an Ambiguous/Underspecified Task (DLM Archival)

**Situation:** Leading the Archival feature for DLM with an unclear PRD on failure handling — unspecified whether to retry, delete, or leave snapshots in the standard tier.

**Task:** Produce a clear plan for failure handling across the scheduler, worker, and API despite vague requirements.

**Action:**
- Mapped every failure point and its downstream effects.
- Analyzed dependencies between API, Lambda, DynamoDB, scheduler, and worker.
- Weighed explicit tradeoffs: cost vs. safety, latency vs. consistency.
- Ran design review meetings with the PM and engineers to close the gaps.

**Result:** Feature launched successfully with predictable failure handling; reliability held >99.9%; customer adoption grew steadily; new engineers onboarded quickly onto the resulting design.

**Good for:** decision with incomplete information, technical judgment & tradeoffs, turning ambiguity into a concrete design — a strong answer for "tell me about a time requirements were unclear."

---

## #dlm-console-ux — Overcoming External Obstacles: DLM Console UX Component

**Situation:** Building the DLM Console required a multi-step component for dynamic workflows, but the existing AWS React component only supported fixed workflows.

**Task:** Deliver a working component that matched the intended UX while getting sign-off from the UX team, who initially pushed a different approach.

**Action:**
- UX team initially suggested reusing the fixed-workflow component.
- Explained why that would hurt the customer experience (empty steps, extra clicks).
- Built a custom multi-step component that handled dynamic workflows correctly.
- Demoed it to the UX team, showing behavior, accessibility, and edge cases explicitly.
- Coordinated with backend engineers to align on the data contract.

**Result:** UX team approved the custom component; multi-schedule workflows worked fully, improving usability; the component was later generalized into a standard version other teams adopted; avoided costly rework by building it correctly upfront.

**Good for:** disagreeing with a decision and changing someone's mind, influence without authority, explaining tradeoffs to a non-technical stakeholder (UX).

---

## #pipeline-fix-vs-bypass — Conflict Resolution: Fix the Pipeline vs. Bypass It (EBS Direct)

**Situation:** A new region build for EBS Direct had a CI/CD pipeline failing during integration tests. A teammate wanted to bypass the check because the task was urgent and the failure looked minor.

**Task:** Decide between moving fast by bypassing the check, or taking the time to properly fix it and protect the reliability of the deployment.

**Action:**
- Investigated the logs and found the root cause was a dependency bug in another team's service.
- Recognized that ignoring it risked breaking future region builds or causing inconsistencies.
- Opened a ticket with the owning team and worked with them on a fix.
- Added extra canary checks and alarms to the pipeline as a longer-term safeguard.
- Kept other work moving in parallel so the urgency didn't stall entirely.

**Result:** Fix landed within the schedule buffer; the region launched on time; deployment stayed stable; the new monitoring caught similar problems before they recurred.

**Good for:** disagreeing with a peer and pushing back, balancing urgency vs. reliability, technical leadership and risk assessment, constructive disagreement/compromise.

---

## #adc-region-mistake — Growth Story: ADC Region Config Hardcoding Mistake

**Situation:** During an AWS region build, set up ADC region configuration managing dependencies between EBS services.

**Task:** Moved faster by hardcoding region-specific dependencies, assuming they'd stay consistent across regions.

**Mistake:** Deploying to a new region caused a build failure because that assumption didn't hold — dependencies were mismatched.

**Action:**
1. Debugged the pipeline logs and traced the failure to the hardcoded logic.
2. Refactored the code to load dependencies dynamically from a shared configuration module.
3. Added a validation check in the CI/CD pipeline to catch outdated/missing configs before deployment.

**Result:** Region builds became fully scalable and less error-prone; region setup time dropped by ~30%; documented the dynamic-configuration process in a runbook for faster onboarding; the system became easier to maintain since new dependencies no longer required code changes.

**Good for:** "biggest mistake you've made," a time you were wrong about a technical bet, growth/self-awareness — this is your strongest genuine-failure story; use it whenever an interviewer explicitly asks for a mistake or a time you were wrong.

---

## #manager-feedback-bigger-picture — Growth Story: Feedback on Thinking Beyond Your Own Tasks

**Situation:** Early in the SDE II role, a manager gave feedback that, as a senior engineer, you shouldn't focus only on your own tasks — you need to think about team impact and the bigger picture.

**Task:** Actively figure out how to broaden your perspective beyond your immediate feature work.

**Action:**
- Asked senior engineers how they approach design discussions and think about system-wide impact.
- Observed how they engaged in meetings — what questions they asked, what connections they made.
- Started joining discussions more actively, including on work that wasn't your own feature.

**Result:** In a design meeting for a new feature, proposed a safer failure-handling approach using retry logic and state transitions; walked the team through it and incorporated their feedback; the team adopted the approach, making the system more reliable. The manager later confirmed observing the shift toward thinking about the project/team, not just individual work.

**Good for:** feedback that was hard to hear, growth & self-awareness, "how has your approach to system design changed."

---

## #recycle-bin-backend-comms — Growth Story: Backend Communication During Recycle Bin Project

**Situation:** Leading frontend development for the Recycle Bin console, a greenfield React project. Manager gave feedback to communicate design dependencies with the backend team more proactively, after starting to build the UI while backend APIs were still being finalized.

**Task:** The backend team worked on a different schedule, causing schema mismatches and integration delays; manager suggested earlier alignment and clearer interface expectations.

**Action:**
- Set up weekly syncs with the backend engineers.
- Created a shared API contract doc defining payload structures and example responses together.
- Built mock APIs so frontend work could progress without waiting on backend deployment.

**Result:** Significantly reduced integration friction; caught schema mismatches early; launched on time. Learned that clear, early communication is as critical as good code in cross-team projects.

**Good for:** feedback that was hard to hear, cross-team collaboration, a time you had to learn/adjust your working style quickly.

---

## #cloud-network-automation — Automated Diagnosis Tool for LB Upgrade/Change Errors (Alibaba Cloud)

**Situation:** As an SRE on the Cloud Network team at Alibaba Cloud, on-call for the load balancer (LB) product and its controller. Errors triggered by LB upgrades/changes were a recurring, messy category of on-call issues — there was no unified SOP and no error-code classification, so every occurrence meant manually digging through logs and communicating back and forth with the owning SDE team to figure out what had actually gone wrong. This meant a large share of these issues had to escalate to SDEs, adding to their on-call load for problems that didn't all strictly need their involvement.

**Task:** Reduce how often LB upgrade/change errors needed to escalate to SDEs — build a way for the SRE team to classify and resolve more of these directly, offloading SDE on-call instead of just absorbing the manual investigation cost every time.

**Action:**
- Started by manually investigating the backlog of past incidents and communicating with SDEs to build up a working understanding of the actual root-cause categories behind these errors, since no classification existed yet.
- Turned that understanding into a defined error-code taxonomy and built an automated diagnosis/analysis tool ("skill") on top of it, so a new incident could be automatically classified against known error patterns instead of a human re-deriving the root cause from scratch every time.
- Rolled the tool out and quickly trained/enabled other SREs on the team to use it, rather than keeping it as a personal workaround.

**Result:** Weekly manual investigation/statistics time dropped from about two days per week to about five minutes; the offload rate (share of LB upgrade/change issues the SRE team could resolve without escalating to SDEs) rose from 50% to 90%.

**Good for:** the SRE→SDE transition narrative's primary evidence story — this is the one to lead with (see `../interview-questions/03-behavioral-and-leadership-questions.md` → "Current-role / career-narrative questions"); "tool/automation to reduce toil"; "identified a problem nobody asked you to solve" (no SOP/classification existed before this); ownership beyond assigned scope; turning ambiguity into a systematic taxonomy; scaling your own impact by enabling teammates rather than just yourself.

---

## Project highlights (for quick framing/context, not full STAR)
- **Archival Feature Launch** — storage archival solution moving infrequently accessed data to lower-cost tiers, cutting customer storage costs and increasing service adoption, with 99.99% availability.
- **Multi-Region Build** — led design and implementation of a multi-region deployment framework for an AWS service, enabling automatic rollout across 5+ regions with built-in health checks and rollback, strengthening global reliability and speeding future feature delivery.
- **DLM Console Migration** — led migration of the DLM Console from GWT to React, modernizing the frontend stack; cut page load time and reduced feature delivery cycles by over 50%.
