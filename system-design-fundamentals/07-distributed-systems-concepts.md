# Core Distributed Systems Concepts

## Why it comes up
These are the building blocks that show up *inside* almost every infra design question — locks, leader election, failure detection, clocks. Interviewers use them to check whether you understand distributed systems are fundamentally about partial failure and unreliable communication, not just "more computers."

## The fundamental problem: partial failure
In a single machine, a function either returns or the machine is down (fail-stop). In a distributed system, a request can be slow, lost, duplicated, or reordered — and **you cannot distinguish "the remote node is slow" from "the remote node is dead" from "the network is down."** This is the root cause of almost every hard distributed systems problem (timeouts are guesses, not truth).

## Failure detection
- **Heartbeats:** periodic liveness pings; absence of N consecutive heartbeats → suspect failure. Tradeoff: short interval = fast detection but more false positives (e.g., from GC pauses or network blips); long interval = slower detection, fewer false positives.
- **Phi Accrual failure detector:** instead of a hard threshold, computes a suspicion level based on historical heartbeat variance — used by Cassandra, Akka.
- **Gossip protocols:** nodes periodically exchange state with random peers; failure/membership info propagates epidemically, scales well without a central coordinator (used in Cassandra, Consul, Serf).

## Leader election
- Needed whenever you want a single coordinator for some duty (avoid split-brain writes, sequence assignment, scheduling).
- Typically built on consensus (Raft leader election, ZooKeeper ephemeral+sequential znodes, etcd lease + campaign API).
- Key property: **only one leader at a time in any given term**, achieved via majority quorum — a node can only become leader if it gets votes from a majority, so at most one node can win any given term.
- After election, the old leader must be prevented from acting (fencing — see below), because it might still think it's the leader (isolated by a network partition, not actually dead).

## Distributed locks
- Used to ensure mutual exclusion across processes/machines (e.g., only one worker processes a given job).
- Built on top of a consistent store (ZooKeeper, etcd, or Redis with careful design — see Redlock controversy: single-Redis-instance locks are fine for efficiency/best-effort locking but not safe as a correctness guarantee under failover, per Martin Kleppmann's critique of Redlock).
- **Fencing tokens:** the critical safety mechanism — every time a lock is granted, issue a monotonically increasing token; the resource being protected must reject any operation presenting an older token than one it's already seen. This closes the "lock holder pauses (GC/swap), lock expires, someone else acquires it, original holder wakes up and still thinks it holds the lock" race — a lock alone (without fencing) is not actually safe against this.
- Lease-based locks (TTL expiry) trade safety for liveness — a crashed holder eventually releases the lock, but you must handle the "paused, not dead" case above.

## Idempotency & exactly-once semantics
See `05-messaging-and-queues.md` for the full treatment — but note it's a general distributed-systems concept, not just a messaging one: any operation that might be retried (client timeout + retry, at-least-once delivery) needs to be safe to apply more than once.

## Logical clocks (ordering without synchronized physical clocks)
- **Lamport timestamps:** a simple counter incremented on each event and on message receipt (taking the max), gives a total order consistent with causality, but concurrent events get an arbitrary tiebreak order.
- **Vector clocks:** one counter per node, lets you detect true concurrency (neither event happened-before the other) rather than just imposing an arbitrary order — used for conflict detection in leaderless replication.
- **Hybrid Logical Clocks (HLC):** combine physical time with logical counters — gives you causally-consistent ordering that's also close to wall-clock time, used in CockroachDB, MongoDB.
- **TrueTime (Google Spanner):** physical clocks with a bounded uncertainty interval (via GPS+atomic clocks), lets Spanner get external (linearizable) consistency across the globe by waiting out the uncertainty window (`commit-wait`).

## Two Generals' Problem / FLP impossibility (know these exist, briefly)
- **Two Generals:** you cannot achieve guaranteed agreement over an unreliable channel with a finite number of messages — foundational reason acks/retries can never give 100% certainty, only arbitrarily high confidence.
- **FLP impossibility:** in a fully asynchronous system (no bound on message delay), no consensus protocol can guarantee both safety and termination if even one node might fail. Practical consensus systems (Raft, Paxos) work around this by using timeouts (partial synchrony assumption) — they're not violating FLP, they're trading guaranteed termination for practical liveness most of the time.

## Idempotent coordination patterns worth naming
- **Leader lease + fencing token** for safe single-writer semantics.
- **Compare-and-swap (CAS) / optimistic concurrency control** for coordination without locks — read a version, write conditionally on the version being unchanged, retry on conflict.
- **Saga pattern** for distributed transactions without 2PC (see storage doc).

## Common follow-ups
- "Your leader election system elected a new leader, but the old leader is still alive and processing requests — how do you prevent it from causing damage?" → fencing tokens; the resource (e.g., storage) must reject writes with a stale token.
- "How would you implement a distributed lock and what goes wrong with a naive implementation?" → naive TTL lock + no fencing → GC-pause-then-double-write race; explain the fix.
- "Two nodes both think they're the leader — how did that happen and how do you prevent it?" → network partition, no majority-quorum requirement for the election, missing fencing on the write path.
