# Reliability & Resilience (SRE Fundamentals)

## Why it comes up
This is where infra interviews diverge most sharply from generic backend interviews — you're expected to design for failure as the normal case, and speak the vocabulary of SLIs/SLOs, blast radius, and graceful degradation.

## SLI / SLO / SLA (know the distinction precisely)
- **SLI (Service Level Indicator):** a measured metric, e.g., "% of requests completed successfully in <300ms."
- **SLO (Service Level Objective):** an internal target for that SLI, e.g., "99.9% over a rolling 30 days."
- **SLA (Service Level Agreement):** an external, often contractual, commitment with consequences (credits/penalties) for missing it — usually looser than the internal SLO, giving buffer.
- **Error budget:** `1 - SLO` — the allowed amount of unreliability. Once burned, the org policy shifts (e.g., freeze risky launches, prioritize reliability work). This reframes reliability from "always be up" (impossible and wasteful) to "spend the budget deliberately."

## Redundancy & failure domains
- **N+1 / N+2 redundancy:** run more capacity than the minimum needed so any 1 (or 2) failures don't cause an outage.
- **Failure domain / blast radius:** the set of things one failure can take down together — a rack, an AZ, a region, a shard, a deploy group. Design so a single failure domain failing doesn't take the whole system down (spread replicas across AZs, shard so one bad shard doesn't affect all tenants, deploy in waves not all-at-once).
- **Active-active vs active-passive:** active-active (all regions serve live traffic) gives better resource utilization and faster failover (no cold-start) but requires handling multi-region write conflicts; active-passive is simpler but wastes standby capacity and failover itself can be risky (rarely-exercised path).

## Graceful degradation
- Shed non-critical features under load before the whole system falls over (e.g., disable recommendations, serve cached/stale data, degrade to read-only) — decide in advance which features are "load-bearing" vs "nice to have."
- **Load shedding:** reject excess requests early (cheaply) rather than letting them queue up and consume resources that starve requests that could have succeeded — often prioritized by request type/customer tier.
- **Circuit breakers, timeouts, retries with backoff+jitter, bulkheads (isolating resource pools per dependency so one slow dependency can't exhaust threads/connections needed by others)** — the standard resilience toolkit, all covered from the networking angle in `06-networking-and-protocols.md`.

## Cascading failure (a favorite interview scenario)
Classic chain: one dependency slows down → callers' threads/connections pile up waiting → callers themselves become slow/exhausted → their callers pile up → outage spreads upstream. Mitigations: aggressive timeouts (shorter than your own SLA budget allows), circuit breakers to fail fast once a dependency is clearly unhealthy, bulkheads/connection pool isolation per dependency, load shedding at the edge, and **not retrying blindly** (a naive retry storm during a partial outage is one of the most common ways minor incidents become major ones — always cap retries and use backoff+jitter).

## Chaos engineering
- Deliberately inject failure (kill instances, add latency, partition the network) in a controlled way to verify the system actually degrades/recovers as designed, rather than assuming it does.
- Start small/scoped (staging, then a small % of prod, game days) and expand as confidence grows; always have a kill switch.
- Tools: Chaos Monkey (random instance termination, pioneered at Netflix), Gremlin, Chaos Mesh (k8s-native).

## Incident response & postmortems
- **Blameless postmortems:** focus on systemic/process causes, not individual blame — people optimize for hiding mistakes if blamed, which destroys the information you need to actually fix things.
- Timeline reconstruction, root cause (often multiple contributing factors, not one single "root" cause — the Swiss cheese model), actionable follow-ups with owners and dates.
- **On-call practices:** clear escalation paths, runbooks for known failure modes, alert on symptoms not causes (see observability doc), avoid alert fatigue (a paged engineer who gets 50 false pages/week will start ignoring pages).

## Capacity planning & disaster recovery
- **RPO (Recovery Point Objective):** how much data loss is acceptable (drives backup/replication frequency).
- **RTO (Recovery Time Objective):** how long recovery is allowed to take (drives whether you need hot standby vs cold backup restore).
- Regular DR drills/failover tests — an untested failover path is not a reliable failover path (this is a very common real-world gap interviewers like probing).
- Capacity headroom planning for both steady growth and sudden spikes (marketing events, viral moments) — include headroom for losing a full redundancy unit (AZ/region) while still serving peak load.

## Common follow-ups
- "Your service depends on a flaky downstream — walk me through your resilience strategy end to end." → timeout tuned below caller's budget → retry with capped backoff+jitter → circuit breaker to fail fast → bulkhead so it doesn't exhaust shared resources → fallback/degraded response → alert on the symptom (elevated error rate/latency to callers), not just the downstream's own health.
- "How do you decide when to page someone at 3am vs let it wait until morning?" → tie directly to SLO/error-budget burn rate and user impact, not raw thresholds.
- "Design a deployment process that limits blast radius." → canary + progressive rollout by wave/region + automated rollback on SLO regression + feature flags to decouple deploy from release.
