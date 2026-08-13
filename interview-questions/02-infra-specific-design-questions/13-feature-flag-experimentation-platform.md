# Design a Feature Flag / Experimentation Platform

## Clarifying questions to ask first
- Simple boolean flags only, or do we need percentage rollouts and A/B experiment assignment (variants + metrics)?
- Targeting rules needed (e.g., "enable for users in EU" or "enable for internal employees")?
- Scale: how many flag evaluations per second across the fleet, and how many distinct flags/rulesets?

## Requirements
### Functional
- Define a flag with a targeting ruleset (on/off, percentage rollout, attribute-based targeting).
- SDK in each service evaluates a flag for a given user/request context and returns a value/variant.
- Support a kill switch that can disable a flag globally, fast.

### Non-functional
- Flag evaluation must add negligible latency to every request that checks it — this is the dominant constraint on the whole design, since flags are often checked many times per request.
- Consistent bucketing: the same user must reliably get the same variant across repeated evaluations (for a stable experience and valid experiment data).
- Flag changes (especially kill-switch disables) must propagate fast — sub-second in the worst case.

## High-level architecture
1. **Management/config service**: where flags and their targeting rules are authored, versioned, and stored (source of truth).
2. **Distribution layer**: pushes the current ruleset out to every service instance's local SDK — via a streaming/pub-sub update channel plus periodic full-sync fallback.
3. **Client SDK**: embedded in each service, holds a local in-memory copy of the ruleset, evaluates flags synchronously against it for every request with no network call in the hot path.
4. **Event/analytics pipeline**: async logging of flag exposures (who saw which variant) for experiment analysis, decoupled entirely from the evaluation path.

## Deep dives

### Low-latency flag evaluation (local SDK evaluation, not a network call per check)
- The single most important design decision: evaluation must never make a network call per check, since a service might check dozens of flags per request. Instead, the SDK holds the full ruleset locally in memory (synced periodically/pushed on change) and evaluates purely in-process — a hash + rule match against local data, sub-microsecond. This is the same "control plane vs data plane" separation seen in service meshes and config systems: the management service is control plane (infrequent, can be slower), evaluation is data plane (every request, must be instant).

### Consistent bucketing
- To ensure the same user always gets the same variant (both across repeated evaluations and stable over the life of an experiment), compute a deterministic hash of `(user_id, flag_id)` — not just `user_id` alone, so the same user gets independently randomized (but individually stable) buckets across different flags/experiments — then map the hash into a bucket range (e.g., 0-99) to compare against the rollout percentage. Because it's a pure deterministic function, no state needs to be stored per user; any instance can compute the same answer for the same user without coordination.

### Propagation latency for flag changes
- Normal flag rollout changes can tolerate a few seconds of propagation delay (eventually consistent is fine — pushed via a pub/sub stream or picked up on the SDK's periodic poll interval). This is the same eventual-consistency-with-bounded-staleness pattern common across most infra config-distribution systems: fast enough to feel real-time, without requiring synchronous consensus on every read.

### Kill-switch requirements
- A kill switch is different in kind from a normal rollout change — it needs to propagate as close to instantly as possible (sub-second), because it exists specifically for "something is on fire, turn this off right now." This usually means a separate, higher-priority push path (e.g., a dedicated pub/sub topic for kill-switch events that bypasses normal batching/polling intervals) rather than relying on the same cadence as routine flag updates.

## Key tradeoffs
- Local in-SDK evaluation is essentially mandatory for latency, but it means every service instance holds a full (or relevant subset of the) ruleset in memory — for orgs with thousands of flags, this can become a non-trivial memory footprint that needs its own scaling consideration (e.g., only syncing flags relevant to that service rather than the entire org's ruleset).
- Deterministic hash-based bucketing requires no per-user state, but it also means you can't easily "grandfather" a specific user into a particular variant after the fact without an explicit override rule layered on top of the hash-based default.

## Failure modes
- Config/management service down: SDKs continue serving from their last-synced local ruleset — flag evaluation for existing flags is completely unaffected, since evaluation never depends on the management service being reachable at request time. Only *new* flag changes fail to propagate until it recovers.
- SDK fails to sync for an extended period: falls back to the last-known ruleset (stale but functional) rather than failing evaluations outright or defaulting to some unsafe assumption — evaluation should always degrade to "keep using what you last knew," never to "error out" or "silently assume off/on."

## Likely follow-ups
- "How would you run an A/B experiment and be confident the split is statistically valid?" → the deterministic hash bucketing already guarantees a stable, effectively-random split; the analytics pipeline needs to log exposure events (not just conversions) so you can correctly compute significance only over users who were actually exposed to each variant.
- "A flag change caused an incident — how do you find out fast which services had it applied and when?" → SDKs should report their currently-loaded ruleset version back to a central place (similar to xDS ACK reporting in a service mesh), giving an audit trail of exactly which version was active where and when.
