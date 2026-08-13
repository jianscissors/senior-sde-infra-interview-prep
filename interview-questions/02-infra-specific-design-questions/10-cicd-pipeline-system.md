# Design a CI/CD Pipeline System (like a simplified GitHub Actions/Jenkins/Buildkite at company scale)

## Clarifying questions to ask first
- Scale: how many teams/repos, how many builds per day, peak concurrency (e.g., everyone pushing right before a release freeze)?
- Do builds need strong isolation (untrusted/multi-tenant code, e.g., open-source contributions) or is it all trusted internal code?
- Do we need to support arbitrary user-defined pipeline steps/plugins, or a fixed set of build/test/deploy stages?

## Requirements
### Functional
- Trigger a pipeline on a code event (push, PR, tag, schedule, manual).
- Run a DAG of build/test/deploy steps, reporting status back to the source system.
- Support secrets injection, artifact passing between steps, and deployment approval gates.

### Non-functional
- Isolation between builds — one team's build must not affect another's (resource contention or security).
- High throughput at peak (many teams push around the same times of day).
- Fast feedback loop — build queue time itself is often the biggest complaint in real systems, so queueing/scheduling design matters as much as execution.

## High-level architecture
1. **Trigger/event ingestion**: webhook or poll from the source control system, normalizes into an internal "build requested" event.
2. **Scheduler**: places queued jobs onto available workers based on resource requirements, priority, and fairness policy.
3. **Worker fleet**: ephemeral, isolated execution environments (containers or short-lived VMs) that pull the job, run pipeline steps, stream logs, and report results.
4. **Artifact store**: object storage for build outputs, passed between pipeline stages and to deployment.
5. **Orchestration/deployment layer**: handles gated rollout to environments after CI passes.

## Deep dives

### Build queue and worker fleet (isolation)
- Each build runs in an ephemeral, single-use container or microVM (e.g., Firecracker-style) that's destroyed after the job completes — this prevents state leakage between builds (a compromised or misbehaving build from one team can't poison the environment for the next build on the same physical worker) and gives a clean, reproducible environment every time.
- For untrusted/multi-tenant workloads (e.g., PRs from external contributors), isolation needs to be stronger than a plain container — VM-level or gVisor-style sandboxing, plus no access to internal network/secrets by default.

### Artifact caching
- Dependency/layer caching (e.g., package manager caches, Docker layer caches, compiler caches) is usually the single biggest lever on build time — cache keyed by a hash of the dependency manifest (e.g., lockfile hash) so cache hits are safe and correctness isn't sacrificed for speed.
- Cache storage needs its own scaling story (object storage, often with a fast local SSD tier co-located with workers for hot caches, since fetching a multi-GB cache over the network from origin storage on every build defeats the purpose).

### Fan-out for parallel test sharding
- Split a large test suite across N workers running in parallel, then aggregate results (pipeline fails if any shard fails). Shard assignment should be timing-aware (historical per-test duration data) rather than naive alphabetical splitting, so shards finish at roughly the same time instead of one long-pole shard dominating total pipeline time.

### Secrets management for pipelines
- Secrets (API keys, deploy credentials) must never be baked into the build image or logged in plaintext output (log scrubbing for known secret patterns is standard). Inject secrets at runtime via a short-lived token scoped to that specific build/job, fetched from a secrets manager, and scoped down to only what that pipeline actually needs (least privilege) — this is the same bootstrapping-problem pattern as any secrets system: the CI job authenticates to the secrets store using an identity tied to the job itself (e.g., a signed job token), not a static credential baked into config.

### Multi-tenant fairness/priority
- Without explicit fairness controls, one team's burst of builds (e.g., a large monorepo CI storm) can starve smaller teams' queues. Use per-tenant queues or weighted fair scheduling, plus priority tiers (e.g., a release-blocking build can preempt routine CI), and cap max concurrent workers per tenant.

### Deployment gating/approval workflows
- Model deployment as its own pipeline stage requiring explicit gates: automated (all tests green, canary metrics healthy) and/or manual (human approval for production). Gates should be pluggable/composable rather than hardcoded, since different environments (staging vs prod) typically need different gate sets.

## Key tradeoffs
- Stronger isolation (full VM per build) costs more startup latency and infra cost than lighter isolation (shared container runtime with cgroup/namespace isolation) — the right choice depends heavily on the trust level of the code being built.
- Aggressive caching speeds up builds dramatically but is a correctness risk if cache keys aren't precise (stale cache silently produces a build that doesn't match the actual source/dependencies) — cache invalidation here is genuinely one of the harder problems in the whole system.

## Failure modes
- Scheduler down: in-flight builds already assigned to workers continue running; new builds queue until scheduler recovers — the scheduler itself should be stateless/quickly-recoverable (state lives in a durable queue/store, not in-process).
- A worker dies mid-build: job is detected as failed via heartbeat timeout and requeued (if idempotent) rather than left hanging indefinitely — pipeline steps should be designed to be safely re-runnable.

## Likely follow-ups
- "How would you speed up a company's slowest, most complained-about pipeline?" → profile where time actually goes (queue wait vs checkout vs dependency install vs test run vs deploy) before optimizing blindly; queue wait time is very commonly the biggest hidden cost, not raw execution time.
- "How do you prevent a secret leaking through pipeline logs?" → automatic log scrubbing for known secret values/patterns, plus scoping secrets narrowly so a leak's blast radius is small even if scrubbing misses something.
