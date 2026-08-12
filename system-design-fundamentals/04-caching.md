# Caching

## Why it comes up
Caching is the cheapest lever for latency and load reduction, and nearly every design benefits from it — but a wrong caching strategy causes subtle correctness bugs (stale data, cache stampedes), which is exactly what interviewers probe for.

## Cache placement
- **Client-side:** browser/mobile cache, closest to user, hardest to invalidate centrally.
- **CDN (edge):** caches static/semi-static content geographically close to users; can also cache API responses with short TTLs.
- **Reverse proxy / API gateway cache:** shared across users for a given endpoint.
- **Application-level (in-process):** fastest, but not shared across instances, causes inconsistency across a fleet; use for small hot datasets.
- **Distributed cache (Redis/Memcached):** shared, network hop but still far faster than the DB; the default answer for "add a cache."
- **Database-level cache:** query cache, buffer pool (mostly automatic, less relevant to design discussion).

## Caching strategies (know all four, and when to use each)
- **Cache-aside (lazy loading):** app checks cache, on miss reads DB and populates cache. Simple, resilient to cache failures (falls back to DB), but first request after eviction is slow and there's a window of staleness.
- **Read-through:** cache itself is responsible for loading from DB on miss (app only talks to cache). Cleaner abstraction, needs cache-provider support.
- **Write-through:** write goes to cache and DB synchronously. Cache always fresh, but write latency = DB write latency.
- **Write-behind (write-back):** write goes to cache, async-flushed to DB later. Fast writes, risk of data loss if cache fails before flush.
- **Write-around:** write goes directly to DB, cache populated only on read. Avoids cache churn for data that isn't re-read soon.

## Eviction policies
- **LRU (Least Recently Used):** most common default.
- **LFU (Least Frequently Used):** better when recency isn't a good popularity proxy.
- **TTL-based expiry:** simplest staleness bound, often combined with LRU.
- **Random / FIFO:** cheap, rarely optimal, sometimes used in specialized/approximate caches.

## Invalidation (the hard part — "there are only two hard things in CS")
- TTL expiry (simplest, accept some staleness window).
- Explicit invalidation on write (delete-then-write or write-then-delete to cache — beware race conditions).
- Versioned/keyed cache entries (e.g., include a version or updated_at in the cache key so old entries just become unreachable rather than needing explicit deletes).
- Event-driven invalidation via CDC/pub-sub when multiple services must invalidate together.

## Failure modes to explicitly name in interviews
- **Cache stampede / thundering herd:** many requests miss simultaneously (e.g., popular key expires) and all hit the DB at once. Mitigate with: request coalescing/single-flight (only one request fetches, others wait), jittered TTLs, early/proactive refresh before expiry, locks around the fetch.
- **Hot key:** one key gets disproportionate traffic, overwhelming a single cache shard. Mitigate with local in-process caching of that key, or replicating/sharding the hot key across multiple cache nodes.
- **Cache penetration:** repeated lookups for keys that don't exist in the DB (e.g., an attack or a bug), bypassing the cache every time. Mitigate with caching negative results (short TTL) or a bloom filter of known-existing keys.
- **Cache inconsistency across regions/instances:** decide if staleness is acceptable; if not, use a single source of truth cache tier or event-driven sync.

## Consistent hashing (deserves its own note, used constantly)
Maps both cache nodes and keys onto a hash ring; a key is owned by the next node clockwise on the ring. Adding/removing a node only remaps the keys between it and its neighbor (~1/N of keys), not the whole keyspace — critical for cache clusters where a full remap would cause a stampede. **Virtual nodes** (each physical node gets many points on the ring) smooth out uneven load distribution.

## Common follow-ups
- "What TTL would you pick and why?" → tie it to how often the underlying data changes and how tolerant the product is of staleness.
- "How do you keep the cache consistent with the DB under concurrent writes?" → discuss the delete-vs-update race (two concurrent writes can leave stale data in cache) and mitigations (versioning, short TTL as a safety net, or accept the tiny window).
- "Design a cache for a system with 100x read/write ratio and hot keys" → cache-aside + local process cache for the hottest keys + consistent hashing with virtual nodes + jittered TTL.
