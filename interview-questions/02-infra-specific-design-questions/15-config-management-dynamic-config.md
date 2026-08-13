# Design a Config Management / Dynamic Configuration System

*(A simplified internal config service backing thousands of services — think an internal LaunchDarkly-adjacent or Consul KV-style system for arbitrary config, not just feature flags.)*

## Clarifying questions to ask first
- Is this pure key-value config, or structured config with schemas/validation?
- How many services/hosts need to watch for changes, and how fast must a change propagate (seconds? sub-second)?
- Do we need staged/canary rollout of config changes, or is it always "push to everyone at once"?
- Who can change config, and do we need an approval/audit trail?

## Requirements
### Functional
- Services can read current config values and be notified of changes without polling on every request.
- Operators can push config changes, target a subset of services/hosts, and roll back.

### Non-functional
- Propagation should be fast (seconds, not minutes) but the system should favor eventual consistency over strict consistency — a config service that sacrifices availability for consistency becomes a fragile dependency for every service in the company.
- Must not become a single point of failure: if the config service is down, dependent services should keep running on their last-known-good config, not crash or hang.

## High-level architecture
1. **Storage layer**: versioned key-value store (each write creates a new version, never mutates in place) — enables instant rollback by pointing back at a prior version.
2. **Distribution layer**: a push/watch mechanism (long-polling or a streaming API) so clients get near-real-time updates instead of polling on a fixed interval, plus a fallback periodic poll for resilience if the stream connection drops.
3. **Client-side SDK**: caches the last-fetched config locally on disk/memory, serves reads from that local cache (never blocks a request on a network call to the config service), and updates it asynchronously in the background.

## Deep dives

### Propagation speed vs consistency
- Because thousands of services are polling/watching, strict consistency (every service having the identical config at the identical instant) is neither achievable nor necessary. Prioritize fast, eventually-consistent propagation: a watch/streaming API (clients hold a long-lived connection and get pushed diffs) is far faster than polling, and the small propagation-lag window is an accepted tradeoff.

### Versioning and rollback
- Every write is a new immutable version with a monotonically increasing version number; the "current" pointer for a config key moves to point at a version. Rollback is then just moving that pointer back to a prior version — no complex undo logic, and it's instant.

### Validation and staged rollout
- A bad global config push is one of the classic causes of major outages (misconfigured global rate limit, wrong feature flag default, bad routing rule). Treat config changes like code deploys: validate against a schema before accepting the write, then **canary** — push to a small percentage of hosts/services first, monitor health signals (error rate, latency) for a bake period, and only then promote to 100%. Provide a fast, automatic rollback trigger tied to health signal regressions, not just a manual one.

### Local caching for graceful degradation
- Every client SDK persists the last-known-good config to local disk. If the config service is unreachable at startup or during operation, the service starts/continues with the cached config rather than failing to boot or hanging on a config fetch. This converts "config service outage" from a company-wide incident into a "config just doesn't update for a while" non-event.

## Key tradeoffs
- Push (watch/streaming) is lower latency but requires holding open connections at scale (connection-count scaling problem on the server side); pure polling is simpler to operate but trades off propagation latency directly against poll frequency and server load.
- Canary rollout adds latency to how fast a config change reaches 100% of hosts, in exchange for containing blast radius — worth it for anything that isn't an emergency kill-switch.

## Failure modes
- Config service fully down: clients keep running on cached config indefinitely; alert loudly on staleness (config age exceeding some threshold) rather than on the service being down per se, since the actual user-facing impact is staleness, not the outage itself.
- A bad config value passes validation but is still logically wrong (e.g., a valid-but-wrong routing rule): this is exactly why canarying with real health-signal monitoring matters more than schema validation alone — schema validation catches malformed input, not bad business logic.

## Likely follow-ups
- "A kill-switch config needs to propagate in under a second — how is that different from normal config?" → bypass the staged-canary path entirely for a designated kill-switch class of config, push directly and immediately to all watchers, accept the higher blast-radius risk because a kill-switch is by definition an emergency stop.
- "How would you audit who changed what, and when?" → since every write is an immutable version with an author and timestamp, the version history *is* the audit log — no separate audit system needed.
