# Study Plan & Progress Tracker

Fill in dates and confidence as you go. Confidence scale: 1 (haven't studied) → 5 (could teach it / handle any follow-up).

## Fundamentals

| Topic | File | Confidence (1-5) | Last reviewed | Notes |
|---|---|---|---|---|
| Scalability & load balancing | `system-design-fundamentals/01-...` | | | |
| Consistency & replication | `system-design-fundamentals/02-...` | | | |
| Storage systems | `system-design-fundamentals/03-...` | | | |
| Caching | `system-design-fundamentals/04-...` | | | |
| Messaging & queues | `system-design-fundamentals/05-...` | | | |
| Networking & protocols | `system-design-fundamentals/06-...` | | | |
| Distributed systems concepts | `system-design-fundamentals/07-...` | | | |
| Observability | `system-design-fundamentals/08-...` | | | |
| Containers & orchestration | `system-design-fundamentals/09-...` | | | |
| Reliability & resilience | `system-design-fundamentals/10-...` | | | |

## Design questions drilled

Track each practice attempt — date, how long you took, and what you'd improve next time. Add rows as you go.

| Question | Date | Time taken | Self-grade | What to improve next time |
|---|---|---|---|---|
| | | | | |

## Priority focus (based on background)
**Current role:** SRE on the Cloud Network team at Alibaba Cloud — on-call across multiple cloud networking components (VPC/SLB/NAT/routing-type products). Targeting a move back to SDE; see the "Current-role / career-narrative questions" section in `../interview-questions/03-behavioral-and-leadership-questions.md` for how to frame this transition, and `#cloud-network-automation` in `star-stories.md` for the (currently placeholder) primary evidence story — fill that in first, it's the highest-leverage gap right now.

**Prior background:** SDE II at AWS EBS Direct (multi-region service builds, CI/CD + infra provisioning via CloudFormation, monthly on-call rotations) and AWS EBS Data Lifecycle Manager (feature design in Java/OOD using Lambda, S3, DynamoDB, CFN; automated unit/integration/canary testing), plus earlier frontend/full-stack work (GWT→React migration, Recycle Bin 0→1 with a full CD pipeline and Game Day simulation testing). Weight practice time accordingly:

- **Play to strength, go deep**: `02-infra-specific-design-questions/18-global-traffic-management-multi-region-failover.md`, `19-distributed-tracing-ingestion-pipeline.md`, `05-metrics-monitoring-pipeline.md` — cloud network on-call gives real, current war-story material here on top of the AWS background; also `system-design-fundamentals/06-networking-and-protocols.md` and `10-reliability-and-resilience.md`, which the deep-dive networking/reliability questions probe hard and where firsthand incident experience beats textbook answers.
- **Also strong from prior AWS work**: `03-distributed-job-scheduler.md` (Lambda-style async orchestration), `15-config-management-dynamic-config.md`, `10-cicd-pipeline-system.md` — these map directly to EBS Direct / DLM work. The Game Day simulation experience is also direct ammunition for the "how do you test that failover actually works" follow-up in the multi-region-failover doc.
- **Storage-adjacent, direct experience**: `07-object-storage-system.md` (hands-on S3 usage, not just adjacent), `08-distributed-file-system.md` — EBS itself is block storage, so "how would EBS's design differ from S3/HDFS" is a very likely probe and one you can answer from firsthand comparison.
- **Testing/reliability practices to lean on**: canary testing experience is directly relevant to deployment-gating discussions in `10-cicd-pipeline-system.md` and canarying-config-changes points in `15-config-management-dynamic-config.md`; on-call rotation experience (both AWS and current Alibaba Cloud) is real material for the on-call/paging behavioral prompts.
- **Gaps to shore up**: pure frontend background means less natural depth on `14-distributed-message-queue.md`, `04-distributed-lock-service.md`, `12-service-mesh-control-plane.md` — these need more deliberate study since they're not close to prior hands-on work.

## Behavioral stories prepared

Full stories in `star-stories.md`; cross-linked to specific prompts in `../interview-questions/03-behavioral-and-leadership-questions.md`.

| Story | Covers which prompts | Polished? (Y/N) |
|---|---|---|
| `#archival-feature` | ownership/scope, complex distributed system, incident-adjacent (monitoring/runbooks) | N |
| `#region-standardization` | self-initiated problem-solving, influence without authority, platform vs point solution | N |
| `#cross-team-unblock` | cross-team collaboration, influence without authority | N |
| `#cicd-bug-outside-scope` | ownership beyond scope, simpler-solution tradeoff, deliberate technical debt | N |
| `#ambiguous-failure-handling` | decision with incomplete information, technical judgment under ambiguity | N |
| `#dlm-console-ux` | disagreement/influence, explaining tradeoffs to non-technical stakeholders, saying no to a stakeholder | N |
| `#pipeline-fix-vs-bypass` | conflict resolution, risky operational call, postmortem, incident prevention | N |
| `#adc-region-mistake` | biggest mistake, technical decision you'd change, incident prevention | N |
| `#manager-feedback-bigger-picture` | hard feedback, how your approach to design has changed | N |
| `#recycle-bin-backend-comms` | hard feedback, cross-team collaboration | N |

**Known gaps (no story yet):** build-vs-buy decision, inheriting a bad system, deciding what NOT to work on, learning a new domain quickly, deprecating a widely-used system, capacity planning gone wrong. Also: expand the "DLM Console Migration" project highlight into a full STAR story for the zero-downtime-migration prompt.

**Gaps resolved by resume detail — draft these into full STAR stories before the interview, they currently only exist as one-line resume bullets, not stories:**
- On-call/paging judgment and sustainability — you ran monthly on-call rotations at EBS Direct as "key contact... addressing issues to minimize downtime." Pick a specific incident from that rotation and turn it into a real STAR story; right now `star-stories.md` has nothing on-call-specific.
- Postmortem — same on-call rotation is likely where a real postmortem example lives; pick one and add it to `star-stories.md`.
- Resilience/failure-injection testing — the Recycle Bin project included "Game Day simulation testing," which is a strong, specific answer for "how do you test failover" or "prevented an incident before it happened" once turned into a full STAR story rather than a bullet.

## Weak areas (update after each practice session)

-

## Target interview date(s)

-
