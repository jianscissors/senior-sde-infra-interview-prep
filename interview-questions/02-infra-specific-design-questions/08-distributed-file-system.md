# Design a Distributed File System (like HDFS/GFS)

## Clarifying questions to ask first
- Workload pattern: write-once-read-many (analytics/batch, like GFS/HDFS's original target) or general-purpose read/write?
- File sizes: optimized for huge files (100MB-multi-GB) or many small files?
- Do clients need POSIX semantics, or is an app-level API (open/read/write/close via a client library) acceptable?

## Requirements
### Functional
- Store very large files, split into fixed-size blocks, replicated across machines.
- Support sequential writes/appends and streaming reads efficiently.

### Non-functional
- Run on commodity hardware where node failure is the norm, not the exception — the system must tolerate frequent individual failures without data loss or operator intervention.
- High aggregate throughput for large sequential I/O, not optimized for low-latency random access to small files.
- Scale to thousands of nodes and petabytes of data.

## High-level architecture
- **Metadata server (NameNode-equivalent)**: holds the filesystem namespace (directory tree, file → block list mapping) in memory for speed, persists a write-ahead log + periodic checkpoint (snapshot) to disk for recovery.
- **Data nodes (chunkservers/DataNodes)**: store fixed-size blocks (e.g., 64-128MB, deliberately large to keep metadata small relative to data and to amortize seek overhead for large sequential reads) as local files, replicated (typically 3x) across nodes and racks.
- **Client**: talks to the metadata server once to resolve a file to its block locations, then talks directly to data nodes for the actual reads/writes — the metadata server is never on the data path, which is essential for throughput at scale.

## Deep dives

### Single metadata server vs distributed metadata
- A single NameNode-style server is simple and gives strong consistency for the namespace, but has two structural problems as the system grows: it's a single point of failure, and the number of files/blocks it can hold is bounded by one machine's memory (metadata is kept in RAM for speed).
- This is precisely why later generations of these systems moved to federated or fully distributed metadata: e.g., HDFS Federation lets multiple independent NameNodes each own a slice of the namespace, and systems like Colossus (Google's GFS successor) distribute metadata across a sharded, replicated store instead of a single process. Naming this evolution explicitly is exactly what a senior-level answer should surface, since "GFS's single master doesn't scale past ~tens of PB" is a well-known, well-documented limitation.

### Block-based storage with replication
- Files are split into blocks, each block replicated (default 3x) onto different data nodes, chosen across failure domains (different racks/AZs) so a single rack failure doesn't take out all replicas of a block.
- The metadata server tracks block→replica-location mappings and periodically receives heartbeats + block reports from data nodes; if a replica count drops below the target (node died), it schedules re-replication from a surviving copy.

### Write-once-read-many optimization
- Because the workload is append-heavy analytics data rather than in-place random-write files, the system optimizes around a simple write pipeline: client writes to one replica, which pipelines the data to the next replica, and so on (a chain, not a fan-out from the client, to maximize the client's outbound bandwidth utilization). Once a block is finalized, it becomes immutable — simplifying consistency drastically versus a POSIX-style random-write filesystem, since concurrent readers never race with an in-place mutation of a completed block.

### Handling the metadata-server SPOF/scale ceiling
- Availability fix: a standby metadata server continuously replays the write-ahead log (or a paired active-standby with shared/replicated edit log storage, e.g., via a small Paxos/ZooKeeper-backed journal) so failover is fast and doesn't lose committed metadata.
- Scale fix: shard the namespace (federation) or move to a distributed, replicated metadata store instead of one process holding everything in RAM — the tradeoff being added complexity in keeping cross-shard operations (e.g., renames across namespace shards) consistent.

## Key tradeoffs
- Large block size (good for sequential throughput, bad for many small files — small files still consume a full metadata entry, wasting the NameNode's limited memory) — a well-known operational problem in real HDFS deployments ("the small files problem").
- Centralized metadata (simple, strongly consistent, but a scale ceiling) vs distributed metadata (scales further, more complex to keep consistent).

## Failure modes
- Data node failure: heartbeat timeout triggers the metadata server to mark its blocks under-replicated and schedule re-replication from surviving copies — self-healing without operator involvement.
- Metadata server failure: standby takes over from the replicated edit log; any writes not yet flushed to the log at the moment of failure are lost, which is why the log write itself must be fsync'd/acknowledged before a client write is considered durable.

## Likely follow-ups
- "How would you support random writes/updates, not just append?" → generally out of scope for this design's assumptions; would require rethinking the immutable-block model entirely (e.g., toward a different system class like a distributed database).
- "How do you avoid all replicas of a popular block ending up on the same overloaded node?" → placement policy explicitly spreads replicas across racks/nodes based on current load and failure-domain diversity, not randomly.
