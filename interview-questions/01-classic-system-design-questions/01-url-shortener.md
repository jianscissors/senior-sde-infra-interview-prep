# Design a URL Shortener (e.g., bit.ly)

## Clarifying questions to ask first
- Custom aliases supported, or only generated short codes?
- Expiration/TTL on links?
- Do we need click analytics (count, geo, referrer)?
- Expected read:write ratio? (Almost always extremely read-heavy.)
- Redirect type: 301 (permanent, cacheable by browsers/CDN) vs 302 (temporary, lets you keep tracking every click)?

## Requirements
**Functional**
- Given a long URL, return a short URL.
- Given a short URL, redirect to the original long URL.
- Optional: custom aliases, expiration, click analytics.

**Non-functional**
- Extremely low redirect latency (this is on the critical path of someone's click).
- High availability for redirects (writes can tolerate brief unavailability more than reads can).
- Read:write ratio is typically 100:1 to 1000:1 — design for read scale.

## Back-of-envelope estimation
- 100M new URLs/day ≈ 1,160 writes/sec average (plan for 5-10x peak).
- Read:write of 100:1 → ~116K redirects/sec average.
- Each mapping row is small (~500 bytes with metadata) → 100M/day × 365 × 5 years × 500B ≈ ~90TB over 5 years — dominated by metadata, not the core key→URL mapping, which is far smaller.
- Short code space: base62 (a-zA-Z0-9), 7 characters gives 62^7 ≈ 3.5 trillion combinations — plenty of headroom.

## High-level architecture
1. **Write path**: Client → API → key generation service → write mapping to storage → return short URL.
2. **Read path**: Client → edge cache/CDN (for hot links) → API → cache (Redis) → storage (fallback on cache miss) → 301/302 redirect.

## Deep dives

### Key generation
- **Counter + base62 encode**: a globally unique counter (or per-shard counter ranges to avoid a single point of contention) encoded to base62. Simple, no collisions, but leaks approximate creation order/volume.
- **Hash (MD5/SHA256) of the URL, truncated**: deterministic, but needs a collision-check-and-retry loop, and the same URL always maps to the same code (fine unless you want distinct codes per user for the same long URL).
- **Pre-generated random key pool**: a background job generates and stores unused base62 keys ahead of time; write path just claims one. Avoids collision checks and contention on a shared counter at request time — this is usually the answer that impresses interviewers, since it decouples key generation latency from the write path entirely.

### Caching strategy for the read-heavy path
- Cache short→long mappings in Redis with an LRU/LFU eviction policy; a small fraction of links (viral links) account for a large fraction of traffic (Zipfian distribution), so cache hit rate will be very high even with a modest cache size.
- Put a CDN/edge layer in front for 301 redirects, since a permanent redirect is safe for browsers and CDNs to cache directly, taking your origin out of the loop entirely for repeat clicks.

### Analytics without slowing the redirect
- Never write analytics synchronously on the redirect's critical path. Fire an async event (to a queue/log) after issuing the redirect, and aggregate downstream (stream processor or batch job) into a separate analytics store.

### Custom aliases & expiration
- Custom aliases need a uniqueness check against the same key space — a unique index on the short code column, insert-or-conflict.
- Expiration: a `expires_at` column checked on read (lazy deletion) plus a periodic background sweep (garbage collection) to reclaim keys and shrink storage — don't rely on the sweep alone, since read-time checks must still enforce it for correctness even if cleanup is delayed.

## Key tradeoffs
- 301 (cacheable, less control) vs 302 (always hits your service, full analytics) — many real systems use 302 specifically to keep click tracking accurate, accepting the extra origin load.
- Counter-based keys are simpler and collision-free but reveal creation order; hash/random keys avoid that at the cost of collision handling.

## Failure modes
- Key-generation service down → pre-generated key pool means writes degrade gracefully (serve from pool) rather than stalling on live generation.
- Cache down → reads fall through to the database; make sure the DB alone can survive a cache-cluster outage at a reduced (but not zero) capacity, or shed load explicitly.
- Database primary down → reads should keep working off replicas; writes queue or fail fast with clear client-facing errors rather than hanging.

## Likely follow-ups
- "How would you handle a link going viral in real time?" → cache warms naturally on first hits; consider pre-warming/pinning known high-traffic keys.
- "How do you shard the mapping storage as it grows?" → shard by hash of the short code for even distribution; the short code itself is a good partition key since lookups are always by short code.
