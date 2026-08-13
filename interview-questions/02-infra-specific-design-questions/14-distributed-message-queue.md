# Design a Distributed Message Queue (like a simplified Kafka)

## Clarifying questions to ask first
- Ordering guarantees needed: global order, or is per-key/per-partition order sufficient?
- Delivery semantics: at-least-once, at-most-once, or exactly-once (via idempotent consumers)?
- Retention: time-based (e.g., 7 days) or size-based, and do consumers need to replay history?
- Expected throughput (messages/sec) and message size, and how many independent consumer groups read the same stream?

## Requirements
### Functional
- Producers append messages to named topics; consumers read messages, optionally as part of a consumer group for parallel, non-overlapping consumption.
- Support replay (re-reading from an earlier offset) within the retention window.

### Non-functional
- High write throughput with durability (no acknowledged message is lost).
- Horizontal scalability of both storage and consumption.
- Bounded end-to-end latency (publish to consumer visibility) even under load.

## High-level architecture
- **Topic → partitions**: each topic is split into partitions, the unit of parallelism and ordering.
- **Broker cluster**: each partition has one leader broker (serves all reads/writes for that partition) and N follower replicas that passively replicate it.
- **Producers** hash a partition key (or round-robin if no key) to pick a partition, then write to that partition's leader.
- **Consumers** are organized into consumer groups; each partition is consumed by exactly one consumer within a group at a time, giving parallelism up to the partition count.

## Deep dives

### Partitioning and ordering-within-partition
- Kafka guarantees order only *within* a partition, not across a topic. All messages for a given key (e.g., a user ID or entity ID) must land on the same partition to see per-key ordering — done by hashing the key modulo partition count. Choosing too few partitions limits parallelism; choosing too many increases per-partition overhead (open file handles, replication traffic, leader-election cost on failure) and, since the hash-to-partition mapping usually depends on partition count, complicates later re-partitioning without breaking key ordering.

### Replication (leader/follower per partition)
- Each partition's leader accepts all writes and serves reads; followers replicate the leader's log by continuously fetching from it. The **in-sync replica (ISR)** set tracks followers that are caught up within an allowed lag; if the leader dies, a new leader is elected only from the ISR set to avoid electing a stale replica and silently losing committed data.

### `acks` / durability tradeoffs
- `acks=0`: producer doesn't wait for any acknowledgment — highest throughput, messages can be silently lost.
- `acks=1`: leader acknowledges after writing to its own local log — good balance, but a message can be lost if the leader dies before followers replicate it.
- `acks=all` (with `min.insync.replicas`): leader waits for a configured number of ISR followers to replicate before acknowledging — strongest durability, highest latency. This is the answer to give when durability is explicitly required.

### Consumer group rebalancing and offset management
- When a consumer joins/leaves a group (or crashes and its heartbeat times out), the group coordinator triggers a **rebalance**, reassigning partitions among the remaining consumers. This is disruptive — consumption pauses for all members during a rebalance — so production designs favor incremental/cooperative rebalancing (reassign only the affected partitions) over a stop-the-world rebalance of everything.
- Each consumer tracks its **offset** (position in the partition) and periodically commits it (to the broker or a compacted internal topic). On restart, the consumer resumes from the last committed offset — committing *after* processing gives at-least-once delivery (possible reprocessing on crash between processing and commit); committing *before* processing gives at-most-once (possible message loss on crash).

### Handling a broker failure without data loss or long unavailability
- Because each partition is replicated across multiple brokers, losing one broker only takes down the partitions it was leading; those partitions fail over to an in-sync follower (leader election, typically coordinated via a metadata/consensus layer like Raft or ZooKeeper-equivalent). Partitions the dead broker was merely following are unaffected. The window of unavailability for the affected partitions is bounded by leader-election time, not full-cluster time.

## Key tradeoffs
- More partitions → more parallelism but more replication/metadata overhead and slower leader election across the whole cluster during a broker failure (more partitions to re-elect).
- `acks=all` gives strong durability at the cost of write latency; most systems make this configurable per-topic based on how critical the data is.

## Failure modes
- Leader dies mid-write: producer gets an error/times out, retries against the newly elected leader; without idempotent producer support, a retry after a successful-but-unacknowledged write can duplicate a message — idempotent producers (dedup via producer ID + sequence number) close this gap.
- A consumer falls far behind (lag grows unbounded): signals either under-provisioned consumers or a slow downstream dependency; monitor consumer lag as a first-class metric, not just broker health.

## Likely follow-ups
- "How would you achieve exactly-once processing end-to-end?" → idempotent producers + transactional writes (atomic multi-partition commits) on the produce side, combined with idempotent consumers (dedup by message ID) on the consume side — true exactly-once delivery is not achievable, but exactly-once *effect* is, via idempotency.
- "How do you handle a topic that needs more partitions after the fact?" → you can increase partition count, but existing key→partition mappings shift, breaking ordering guarantees for existing keys going forward; plan partition count for peak expected parallelism up front.
