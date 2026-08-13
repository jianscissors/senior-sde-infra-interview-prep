# Design a Distributed Cache (like a simplified Redis Cluster/Memcached)

## Clarifying questions to ask first
- Cache-aside (app manages reads/writes to cache + DB) or is the cache itself the system of record for some data?
- What's the expected key/value size distribution, and total working set size?
- Do we need cross-key transactions/atomic multi-key ops, or is single-key access sufficient?
- Read:write ratio, and how latency-sensitive are callers (p99 budget)?
- Do we need strong consistency for reads-after-write, or is eventual/staleness acceptable?

## Requirements
### Functional
- GET/SET/DELETE on keys, with TTL support.
- Support for a working set larger than any single node's memory (hence "distributed").

### Non-functional
- Sub-millisecond p99 for in-memory hits.
- Horizontal scalability as working set/traffic grows.
- Survive individual node failure without losing all data for the keys it owned, and without a long client-visible outage.

### Back-of-envelope estimation
- 1TB working set, nodes with 64GB usable memory → ~16-20 nodes minimum (leave headroom, don't run at 100% memory).
- 500K ops/sec target, single node handling ~50-100K ops/sec → confirms roughly the same node count from the throughput side, a useful sanity check.

## High-level architecture
Client (thin client library or a proxy layer) → consistent-hash ring mapping key → shard → primary node (+ N replicas per shard) → in-memory store with periodic/async persistence for warm restarts.

## Deep dives

### Sharding via consistent hashing
- Hash each key onto a ring; each node owns an arc of the ring. Adding/removing a node only reshuffles the keys on the adjacent arc, not the whole keyspace — this is the whole point versus modulo hashing (`hash(key) % N`), where changing N remaps almost every key.
- **Virtual nodes**: give each physical node many points on the ring (e.g., 100-256 virtual nodes) so that (a) load spreads evenly even with few physical nodes, and (b) when a node fails, its load is spread across many other nodes instead of dumping it all onto one neighbor.

### Replication for durability/availability
- Each shard has a primary + 1-2 replicas. Writes go to primary, async-replicate to replicas (async keeps write latency low; sync would guarantee no data loss on failover but adds latency to every write).
- On primary failure, a replica is promoted (via a coordination service like ZooKeeper/etcd, or a gossip-based failure detector). Because replication was async, some very recent writes may be lost on failover — state this tradeoff explicitly, it's usually acceptable for a cache (source of truth lives in the DB) but would not be for a primary datastore.

### Eviction policy
- LRU is the common default; LFU better resists cache pollution from a single burst of one-time reads. Approximate LRU (sampling a few keys and evicting the oldest, rather than maintaining a perfect linked list) is what Redis actually does, trading a little accuracy for much lower overhead.
- TTL-based expiration runs alongside eviction — lazy (check on access) plus active background sweep, same pattern as the URL shortener's expiration design.

### Hot keys
- Consistent hashing balances key *count* evenly but not necessarily *traffic* — one viral key can overwhelm the single node/shard that owns it. Mitigate with client-side local caching of hot keys, or explicit key replication/sharding of a single hot key across multiple nodes with a random suffix, aggregated by the client.

### Client protocol: thin client vs proxy layer
- **Thin client**: app talks directly to the correct shard, using a client-side copy of the hash ring/routing table. Lowest latency (no extra hop), but every client needs to stay in sync with cluster topology changes.
- **Proxy layer** (e.g., Twemproxy, Envoy): clients talk to a stateless proxy that does the routing. Simplifies clients and centralizes topology/failover logic, at the cost of an extra network hop and the proxy tier itself needing to scale and stay highly available.

## Key tradeoffs
- Async vs sync replication: availability/latency vs durability on failover.
- Thin client vs proxy: latency and client complexity vs operational simplicity and centralized control.

## Failure modes
- Shard primary dies mid-write: client sees a timeout/error, retries against the newly promoted replica once failover completes; app-level idempotency matters if the write was a non-idempotent increment.
- Whole cache cluster down: application must degrade gracefully — read-through to the database directly, at reduced throughput, rather than cascading failure from every request suddenly hitting an unprepared DB at full traffic.

## Likely follow-ups
- "How do you handle resharding when adding capacity without a full cache-cold-start?" → consistent hashing minimizes remapped keys, and you can dual-write/gradually migrate rather than cutting over all at once.
- "What happens on a network partition between a primary and its replicas?" → classic split-brain risk; needs a coordination service or quorum-based promotion to avoid two nodes believing they're both primary.
