# Design a Rate Limiter

## Clarifying questions to ask first
- Rate limit per user, per IP, per API key, or per endpoint?
- Single server or distributed (multiple app servers behind a load balancer)?
- Hard limit (reject over the limit) or soft/throttle (queue or degrade)?
- What should happen to the rejected request — HTTP 429 with `Retry-After`, silently drop, or queue?

## Requirements
**Functional**
- Allow up to N requests per time window per client key.
- Reject (or throttle) requests exceeding the limit with a clear signal to the client.

**Non-functional**
- Rate-limiting check itself must add negligible latency (single-digit ms).
- Must work correctly across many app server instances (distributed correctness), not just per-process.
- Should be resilient to the rate limiter's own storage being briefly slow/unavailable.

## Algorithms (the core of this question)
- **Fixed window counter**: increment a counter per time bucket (e.g., per minute). Simple, O(1) memory, but allows burst at window boundaries (2x the limit right at the edge between two windows).
- **Sliding window log**: store a timestamp per request, count requests in the trailing window. Accurate, but memory grows with request volume.
- **Sliding window counter**: weighted average of current and previous fixed window counts, approximating a sliding log with fixed-window memory cost. Good practical default — accurate enough, cheap enough.
- **Token bucket**: bucket refills at a fixed rate up to a capacity; each request consumes a token. Naturally allows controlled bursts up to the bucket size, then smooths to the refill rate. Most commonly the "right" answer when the interviewer wants to see burst tolerance discussed.
- **Leaky bucket**: requests enqueue into a fixed-rate-draining queue; smooths bursts into a constant output rate but adds queuing latency.

## Where it lives
- **Client-side**: advisory only, not enforceable — a well-behaved client can self-throttle, but you can't rely on it for protection.
- **API gateway / edge**: centralizes the logic, protects everything behind it, but is a shared dependency for every service.
- **Per-service sidecar**: decentralized, no single point of failure for the whole system, but each instance needs to coordinate state with the others for a global limit.

## Deep dives

### Distributed rate limiting
- **Shared counter in Redis**: atomic `INCR` + `EXPIRE` (or a Lua script combining check-and-increment atomically to avoid a race between check and increment). Simple, accurate, but Redis becomes a single dependency in the hot path — must be fast and highly available (cluster it).
- **Local approximate counters with periodic sync**: each app instance keeps a local counter and syncs/reconciles with a central store every N seconds. Much lower latency and no per-request network hop, but the effective limit can overshoot by up to (number of instances × sync interval's worth of requests) — acceptable for most abuse-prevention use cases, not for a hard billing-enforcement limit.

### Fairness across tenants
- A single global limiter can let one noisy tenant starve others sharing the same limited resource. Solve with per-tenant buckets (not one global bucket), and/or weighted fairness so no tenant's burst can starve another's baseline traffic.

## Key tradeoffs
- Accuracy (sliding log, synchronous Redis check) vs latency/scalability (local approximate counters). Most infra rate limiters accept approximate limits in exchange for lower latency and no single point of failure.
- Fixed window is the simplest to implement and explain but has a real boundary-burst flaw — an interviewer will usually probe for this if you propose it as your final answer.

## Failure modes
- **Fail-open vs fail-closed** when the limiter's storage is down is the single most important design decision to state explicitly: fail-open (allow all traffic through) protects availability but removes protection exactly when the system may be under stress; fail-closed protects backend systems but can turn a limiter outage into a full outage for legitimate users. Most production systems fail-open for abuse-prevention limiters and fail-closed only for hard business-logic limits (e.g., paid-tier quotas).

## Likely follow-ups
- "How would you rate-limit by multiple dimensions at once (per-user AND per-IP AND global)?" → run multiple independent checks, reject if any one is exceeded; keep them as separate keys/buckets, not combined into one.
- "The Redis check adds latency to every request — how do you reduce it?" → local caching with periodic sync, or a sidecar co-located with the app to cut network hops.
