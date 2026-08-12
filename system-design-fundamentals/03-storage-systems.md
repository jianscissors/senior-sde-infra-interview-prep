# Storage Systems

## Why it comes up
Infra roles frequently touch the storage layer directly (building or operating databases, object stores, or the systems that sit next to them). You need to know how storage engines work internally, not just "use Postgres."

## Partitioning (sharding)
- **Range-based:** partition by key ranges. Good for range scans, but risks hot spots on sequential keys (e.g., timestamps).
- **Hash-based:** partition by hash(key). Even distribution, but loses range-scan locality.
- **Consistent hashing:** minimizes reshuffling when nodes are added/removed (only ~1/N keys move instead of nearly all). Virtual nodes smooth out load distribution across heterogeneous hardware.
- **Directory-based:** a lookup service maps key → shard, allows flexible rebalancing at the cost of an extra hop/SPOF risk (mitigate with caching + replication of the directory).
- **Rebalancing:** avoid full-cluster reshuffles; move data gradually; watch for hot shards from skewed keys (celebrity/hot-key problem) — mitigate with key salting or dedicated hot-key handling.

## Storage engine internals
- **B-trees:** used by most relational DBs (Postgres, MySQL/InnoDB). Good read performance, in-place updates, page-based.
- **LSM-trees (Log-Structured Merge trees):** used by Cassandra, RocksDB, LevelDB, HBase. Writes go to an in-memory memtable + append-only WAL, flushed to immutable sorted SSTables, merged via compaction. Optimized for write-heavy workloads; reads may need to check multiple SSTables (mitigated with bloom filters).
- **Write-ahead log (WAL):** durability mechanism — write to an append-only log before applying to the in-memory/on-disk structure, enables crash recovery.
- **Bloom filters:** probabilistic set-membership check, no false negatives, used to skip SSTables that definitely don't contain a key.

## SQL vs NoSQL (frame as tradeoffs, not "NoSQL is better at scale")
- **Relational (SQL):** strong schema, ACID transactions, joins, secondary indexes. Scales vertically easily, horizontally with more effort (sharding, or use distributed SQL like Spanner/CockroachDB/YugabyteDB which layer consensus + 2PC over sharded ranges).
- **Key-value:** simplest model, scales horizontally naturally (DynamoDB, Redis).
- **Document:** flexible schema, good for nested/denormalized data (MongoDB).
- **Wide-column:** huge write throughput, sparse schema, good for time-series/logging (Cassandra, HBase, Bigtable).
- **Graph:** relationship-heavy queries (Neo4j).
- **NewSQL / distributed SQL:** ACID + horizontal scale via sharding + consensus (Spanner uses TrueTime for external consistency; CockroachDB uses hybrid logical clocks).

## Transactions
- **ACID:** Atomicity, Consistency, Isolation, Durability.
- **Isolation levels:** Read Uncommitted → Read Committed → Repeatable Read → Serializable. Know what anomaly each one prevents (dirty read, non-repeatable read, phantom read).
- **Distributed transactions:** Two-Phase Commit (2PC) — blocking, coordinator SPOF; Saga pattern — sequence of local transactions with compensating actions for rollback, used widely in microservices since 2PC doesn't scale well across services.
- **MVCC (Multi-Version Concurrency Control):** readers don't block writers by keeping multiple versions of a row (Postgres, MySQL InnoDB). Enables snapshot isolation cheaply.

## Object storage (S3-like systems)
- Flat namespace (not a real filesystem), objects addressed by key, strong read-after-write consistency is now standard (S3 since 2020).
- Built on top of: metadata service (key → location mapping, often a distributed KV/DB) + data nodes storing erasure-coded or replicated chunks.
- Durability via **erasure coding** (like RAID but distributed — e.g., 10 data shards + 4 parity shards tolerate 4 failures with less overhead than 3x replication) vs simple replication (3x copies, more storage overhead, faster rebuild).
- Multipart upload for large objects, versioning, lifecycle policies (tiering to cold storage).

## Common follow-ups
- "Why would you pick LSM over B-tree for this workload?" → write-heavy, append-style ingestion (logs, metrics, time-series) favors LSM; read-heavy with lots of point lookups/updates favors B-tree.
- "How do you handle a hot partition?" → key salting/sharding suffix, splitting the partition, caching in front, read replicas for that shard.
- "How do you do a live resharding without downtime?" → dual-write or change-data-capture (CDC) to backfill, cut over reads once caught up, verify, then stop writing to the old shard.
