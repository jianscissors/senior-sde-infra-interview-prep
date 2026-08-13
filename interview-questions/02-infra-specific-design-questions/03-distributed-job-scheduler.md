# Design a Distributed Job Scheduler (like a simplified Kubernetes Scheduler, or Airflow/cron-at-scale)

## Clarifying questions to ask first
- One-off jobs, recurring (cron-style) jobs, or DAGs of dependent jobs?
- Are jobs idempotent, or does the scheduler need to guarantee exactly-once execution?
- What's the failure/retry policy expected — automatic retries, backoff, dead-letter after N failures?
- Scale target: how many jobs/sec need to be scheduled, and how many concurrent workers?

## Requirements
### Functional
- Accept job submissions (one-off, cron, or DAG), assign them to workers, track completion.
- Support priorities and per-tenant fairness.
- Support dependency graphs (DAG execution) where a job only runs after its upstream jobs succeed.

### Non-functional
- No double-scheduling of the same job (at-most-once assignment) even with a scheduler failure mid-decision.
- Workers can die mid-job; the system must detect and recover without losing the job or silently double-running it unsafely.
- Fair resource allocation across tenants sharing the same worker pool.

## High-level architecture
Job submission API → durable job queue/store → scheduler (assigns jobs to workers based on capacity/priority) → worker pool (pulls or receives assignments, executes, reports status) → job state store (tracks pending/running/succeeded/failed).

## Deep dives

### Job queue and worker pool model
- **Push model**: scheduler actively assigns a job to a specific worker. Gives the scheduler tight control over placement (useful for bin-packing/priority), but the scheduler needs an up-to-date view of every worker's capacity.
- **Pull model**: workers poll a shared queue and claim jobs themselves (e.g., using a visibility timeout like SQS). Simpler, self-balancing (idle workers naturally pick up more work), but less precise control over placement/priority ordering.
- Most production systems (Kubernetes scheduler is push-based; many job queues are pull-based) pick based on how much placement precision they need — a scheduler with node affinity/resource-fit rules almost has to be push-based to make an informed decision.

### Leader election for the scheduler itself
- Run multiple scheduler replicas for availability, but only one should be actively assigning jobs at a time to avoid double-scheduling. Use a consensus-backed lock (etcd/ZooKeeper lease, or Raft-based leader election) — the active leader holds a renewable lease; if it dies or stalls, the lease expires and a standby takes over.
- This is exactly the distributed-lock-service pattern: the lease needs a fencing token so that if the old "leader" was just paused (GC, network blip) and wakes up believing it's still leader, its stale writes get rejected rather than corrupting state.

### Handling worker failure mid-job
- Workers send periodic heartbeats while executing; scheduler tracks a lease/TTL per running job. If the heartbeat stops, the scheduler assumes the worker died and reschedules the job onto another worker after the lease expires.
- This requires jobs to be **idempotent** or the system needs at-least-once semantics with dedup — the worker that "died" might actually still be running (network partition, not a crash), so you can get two workers executing the same job. Either design jobs to be safely re-runnable, or have workers fence themselves (check they still hold the lease before committing side effects).

### Priority and fairness across tenants
- Simple priority queues can starve low-priority tenants indefinitely. Use weighted fair queuing or a priority queue with aging (a job's effective priority increases the longer it waits) so nothing starves forever.
- Per-tenant quotas/concurrency caps prevent one tenant from monopolizing the whole worker pool even at equal priority.

### DAG execution
- Model as a directed acyclic graph; a job becomes eligible to run only when all its upstream dependencies have succeeded. Track this with a per-DAG-run state machine, and support partial re-runs (retry just the failed node and its downstream, not the whole DAG) — this is the actual hard part interviewers probe on, not just "detect the graph has no cycles."

### Backfill/catch-up semantics
- For cron-style recurring jobs, define explicitly what happens if the scheduler was down when a scheduled time passed: catch up and run all missed occurrences, run only the most recent missed one, or skip entirely. This is a real footgun in systems like Airflow if not made explicit and configurable per job.

## Key tradeoffs
- Push vs pull worker model: placement precision vs simplicity/self-balancing.
- At-least-once (safe on failure, needs idempotency) vs attempting exactly-once (much harder distributed guarantee, usually not worth the complexity — most systems build at-least-once + idempotent jobs instead).

## Failure modes
- Scheduler leader dies mid-assignment: standby takes over via lease expiry; any job the old leader "assigned" but didn't durably record needs a recovery pass (reconcile in-flight assignments against actual worker state on takeover).
- Worker fleet-wide outage: jobs pile up in the queue; scheduler should expose backlog depth so autoscaling (or a human) can react, rather than silently queuing forever.

## Likely follow-ups
- "How do you prevent the same job from running twice if a worker's heartbeat is just delayed, not actually dead?" → fencing tokens / lease-based ownership checks before the worker commits any side effect.
- "How would you prioritize a large backlog of jobs from one tenant without starving everyone else?" → weighted fair queuing plus per-tenant concurrency caps.
