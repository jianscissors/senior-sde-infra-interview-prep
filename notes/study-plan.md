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
Background: SDE II at AWS EBS Direct (multi-region deployment, operational excellence) and AWS EBS Data Lifecycle Manager (feature design, Java, Lambda, DynamoDB), plus earlier frontend/full-stack work (GWT→React migration, Recycle Bin 0→1). Weight practice time accordingly:

- **Play to strength, go deep**: `02-infra-specific-design-questions/18-global-traffic-management-multi-region-failover.md`, `03-distributed-job-scheduler.md` (Lambda-style async orchestration), `15-config-management-dynamic-config.md`, `10-cicd-pipeline-system.md` — these map directly to real EBS Direct / DLM work, so interviewers will push hardest here expecting real depth.
- **Storage-adjacent, worth extra reps**: `07-object-storage-system.md`, `08-distributed-file-system.md` — EBS itself is block storage, so "how would EBS's design differ from S3/HDFS" is a very likely probe.
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

**Known gaps (no story yet):** paging/alert-noise story, build-vs-buy decision, inheriting a bad system, deciding what NOT to work on, learning a new domain quickly, deprecating a widely-used system, capacity planning gone wrong, on-call sustainability. Also: expand the "DLM Console Migration" project highlight into a full STAR story for the zero-downtime-migration prompt.

## Weak areas (update after each practice session)

-

## Target interview date(s)

-
