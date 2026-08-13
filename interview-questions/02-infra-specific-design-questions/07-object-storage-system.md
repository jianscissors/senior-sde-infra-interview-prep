# Design an Object Storage System (like a simplified S3)

## Clarifying questions to ask first
- Object size range — mostly small (KBs) or do we need to support multi-GB/TB objects?
- Durability target (e.g., 11 nines) vs availability target — these drive different design knobs.
- Read-after-write consistency required, or is eventual consistency acceptable?
- Do we need versioning, lifecycle policies (tiering to cold storage), or encryption at rest?

## Requirements
### Functional
- PUT/GET/DELETE objects by key within a bucket namespace.
- Support objects from a few bytes up to multi-TB (via multipart upload).
- List objects by prefix.

### Non-functional
- Very high durability (data must essentially never be lost) — this dominates the design more than latency does.
- High availability for reads and writes.
- Strong read-after-write consistency for new objects (S3's actual bar today).
- Massive scale: exabytes of data, trillions of objects, near-linear horizontal scalability.

## High-level architecture
Split into two independent planes:
1. **Metadata/control plane**: maps `(bucket, key) → object location, version, size, ACL, etc.` Backed by a distributed, strongly consistent key-value/database layer (sharded by bucket+key hash).
2. **Data plane**: stores the actual bytes, chunked into fixed-size blocks/objects spread across a large fleet of storage nodes, independent of the metadata layer's request path once location is resolved.

Separating these planes lets each scale on its own axis — metadata is small and needs strong consistency and low-latency lookups; data is huge and needs raw throughput and durability, not transactional semantics.

## Deep dives

### Metadata service vs data plane separation
- The metadata layer is a sharded index: `key → {list of chunk IDs, storage node locations, checksum, version}`. It must be strongly consistent (a client must never be told an object exists and then fail to find its data).
- The data plane nodes are dumb block stores — they don't know about buckets or keys, just chunk IDs and bytes. This separation is what lets you scale petabytes of storage without every storage node needing to participate in metadata consensus.

### Durability: erasure coding vs replication
- **Replication** (e.g., 3x copies across failure domains/AZs): simple, fast to reconstruct (just read a healthy copy), but 3x storage overhead.
- **Erasure coding** (e.g., Reed-Solomon, splitting an object into k data shards + m parity shards, tolerating the loss of any m shards): storage overhead drops to roughly (k+m)/k instead of 3x — e.g., 10 data + 4 parity shards survives any 4 node failures at only 1.4x overhead. Costs more CPU on reconstruction and adds latency to reads that hit a degraded shard set. Real systems use replication for hot/small objects (fast path) and erasure coding for larger, colder objects where the storage savings dominate.

### Consistency model
- Strong read-after-write: once a PUT is acknowledged, every subsequent GET (from anywhere) must see it. Achieved by only acknowledging the client after the metadata write is durably committed (via consensus, e.g., Raft/Paxos over the metadata shard) and a quorum of data replicas/shards confirm the write.
- Overwrites and deletes need the same discipline — a classic bug is acknowledging a PUT before all required replicas are durable, then losing the object on an immediate node failure.

### Multipart upload for large objects
- Client splits a large object into parts (e.g., 100MB each), uploads them in parallel/out of order, each part gets its own checksum; the service assembles/commits the full object only after a final "complete multipart upload" call lists all part IDs. This allows resumable uploads (retry just the failed part, not the whole object) and parallel throughput for very large files.

### Hot object handling
- A single viral object (e.g., a public dataset or trending image) can create a hotspot on the few storage nodes holding its shards. Mitigate with: replicating hot objects to additional nodes dynamically based on access patterns, fronting reads with a CDN/cache layer, and spreading an object's shards across enough distinct nodes that no single node saturates.

### Lifecycle / tiering
- Policies to auto-transition objects between storage classes (hot SSD-backed → warm → cold/archive, e.g., tape-like Glacier-style storage) based on age or access frequency, run as an async background scanner rather than inline on every read/write — trades retrieval latency for storage cost on infrequently accessed data.

## Key tradeoffs
- Erasure coding saves storage cost significantly but costs CPU and can raise tail latency on reconstruction reads — most systems use it selectively.
- Strong consistency requires quorum writes/consensus overhead on every mutation, but is what modern object stores commit to because eventual consistency created a long history of surprising client bugs.

## Failure modes
- Storage node failure: detected via heartbeats, background scrubber re-replicates/re-encodes the lost shard's data from surviving shards to restore full redundancy — this "self-healing" background repair is core to hitting high durability targets over the object's lifetime, not just at write time.
- Metadata shard leader failure: consensus group elects a new leader; brief write unavailability for that shard, reads may still be served from followers depending on consistency requirements.

## Likely follow-ups
- "How do you delete an object that's erasure-coded across many nodes?" → tombstone in metadata immediately (so it's invisible to reads), reclaim the physical shards asynchronously via garbage collection.
- "How would you support versioning?" → key includes a version ID; metadata keeps a version history per key instead of overwriting in place, and a "latest" pointer is updated on each write.
