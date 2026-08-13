# Design a Rate Limiter Service Used by Hundreds of Internal Microservices

## Clarifying questions to ask first
- Is this a shared network service every microservice calls, or a library/sidecar embedded in each service?
- Do limits need to be exact (hard billing enforcement) or approximate (abuse/overload protection)?
- Single-region or multi-region — do limits need to be globally consistent across regions?
- What's the call volume this limiter itself needs to sustain (it's now a dependency for hundreds of services)?

## Requirements
### Functional
- API to check-and-consume a rate limit for a given (tenant, resource) key, returning allow/deny.
- Configurable limits per tenant/service, updatable without redeploying callers.

### Non-functional
- Must not become the slowest link in any caller's request path — target low-single-digit-ms p99 added latency.
- Must itself be highly available, since hundreds of services now depend on it.
- Must scale to the combined QPS of every service that calls it, likely far higher than any single service's own traffic.

## High-level architecture
Every internal service → rate limiter client SDK (embeds fail-open/fail-closed policy and local caching) → rate limiter service (stateless API layer) → shared counter storage (Redis cluster, sharded by limit key).

## Deep dives

### API design as a shared service
- Expose a simple `check(key, cost) -> {allowed, remaining, reset_at}` API, versioned, with a client SDK per language so every team gets the fail-open/retry/timeout behavior right by default instead of reimplementing it (and getting it wrong) per service.
- Limits themselves live in a config store (see the config-management design) so product/infra teams can tune them without a deploy — the rate limiter service watches for config changes and hot-reloads.

### Algorithm choice: sliding window counter
- At this shared-service scale, a sliding window counter (weighted blend of current and previous fixed-window counts) is the sweet spot: far cheaper than a sliding log (no per-request timestamp storage) while avoiding the fixed-window's boundary-burst flaw. Token bucket is the alternative if callers explicitly need burst tolerance above a steady rate.

### Storage for counters
- **Redis with atomic `INCR` + `EXPIRE`** (or a single Lua script doing check-and-increment atomically): strongly accurate, but every check is a network round trip to Redis, and Redis becomes the single most heavily-hit dependency in the whole company's request graph.
- **Local approximate counting with periodic sync**: each rate limiter instance keeps in-memory counters and reconciles with central storage every 1-2 seconds. Removes Redis from the hot path of every single check, at the cost of allowing limits to overshoot by roughly (number of limiter instances × sync interval's request rate) — usually an acceptable tradeoff for infra-level protection, not for something like a paid-tier hard quota.

### Multi-region consistency
- Requiring a globally strongly-consistent counter across regions means every check crosses a WAN — unacceptable latency. Standard answer: each region keeps its own local counter against a per-region slice of the global limit (e.g., divide the global limit across regions by expected traffic share), accepting that the true global rate may briefly exceed the nominal limit during regional traffic shifts. This is explicitly the same "approximate is fine" tradeoff as the local-counter case, just at a coarser granularity.

### Fail-open vs fail-closed
- This is the single most important decision to call out explicitly for a *shared* limiter: if the limiter service itself is degraded, hundreds of services depend on its fail behavior simultaneously. Fail-open (allow all traffic through) is almost always correct here — the blast radius of "rate limiting briefly disabled" is much smaller than "every dependent service starts failing because their rate-limit check timed out."

## Key tradeoffs
- Central Redis-backed accuracy vs local-counter latency/availability — at hundreds-of-services scale, most real systems choose local counters with periodic sync.
- A shared network service (simpler ops, one thing to scale) vs an embedded library (no extra network hop, but versioning/upgrading logic across hundreds of services is much harder).

## Failure modes
- Limiter service down → SDK fails open by default, logs/alerts loudly so it gets fixed quickly, since "silently disabled" for too long defeats the point of having it.
- Storage (Redis) down but service up → service falls back to local-only counting until storage recovers, rather than blocking every check on a dead dependency.

## Likely follow-ups
- "How do you avoid this becoming a single point of failure for the whole company?" → deploy it as a highly-available cluster with fail-open clients, and make sure the SDK has aggressive timeouts so a slow limiter never cascades into slow callers.
- "How would you roll out a new limit without breaking a service that suddenly starts getting throttled?" → staged/canary rollout of config changes, plus dashboards/alerts on rejection rate spikes right after a limit change.
