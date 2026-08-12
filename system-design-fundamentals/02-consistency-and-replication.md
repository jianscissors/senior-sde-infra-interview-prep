# Consistency, Replication & Consensus

## Why it comes up
The single biggest differentiator between junior and senior system design answers: senior candidates explicitly name the consistency model they're choosing and why, rather than hand-waving "we replicate the data."

## CAP theorem (and why it's overused)
Under a network **P**artition, you choose **C**onsistency or **A**vailability. In practice:
- CAP only applies during partitions — most of the time you have all three.
- Use **PACELC** instead when discussing tradeoffs: if Partitioned, choose A or C; Else (normal operation), choose Latency or Consistency. This is the framing senior interviewers actually want to hear.

## Consistency models (strongest to weakest)
- **Linearizability (strict consistency):** every read sees the latest write, as if there were a single copy of the data with a global real-time order. Expensive — needs consensus per operation.
- **Sequential consistency:** all operations appear in some total order consistent with each process's program order, but not necessarily real-time order.
- **Causal consistency:** operations that are causally related are seen in the same order by everyone; concurrent (unrelated) ops may be seen in different orders.
- **Read-your-writes / monotonic reads:** session-level guarantees — a client always sees its own writes, and never sees time go backwards.
- **Eventual consistency:** given no new writes, replicas converge eventually. Weakest, cheapest, most available.

Rule of thumb for interviews: pick the *weakest* model that satisfies the product requirement, and justify it (e.g., "view counts can be eventually consistent; account balances need linearizability or at least strong consistency with idempotent writes").

## Replication strategies
- **Single-leader (primary-replica):** all writes go to the leader, reads can go to replicas (with possible staleness). Simple, but leader is a bottleneck/SPOF until failover.
- **Multi-leader:** writes accepted at multiple nodes, async replication between them. Needs conflict resolution (see below). Good for multi-region write availability.
- **Leaderless (quorum-based, e.g., Dynamo-style):** writes/reads go to W and R nodes out of N replicas; consistency tunable via `W + R > N`. Handles partial failures gracefully.

### Sync vs async replication
- Sync: leader waits for replica ack before confirming write → strong durability, higher latency, availability tied to replica health.
- Async: leader confirms immediately, replicates in background → low latency, risk of data loss on leader failure.
- Semi-sync: wait for at least one replica → middle ground.

## Conflict resolution (multi-leader / leaderless)
- **Last-write-wins (LWW):** simple, can silently drop data; needs synchronized clocks or logical clocks.
- **Vector clocks / version vectors:** detect concurrent (conflicting) writes explicitly.
- **CRDTs (Conflict-free Replicated Data Types):** data structures designed so concurrent updates merge deterministically without coordination (counters, sets, LWW-registers, sequences for collaborative text).
- **Application-level merge:** surface the conflict to the app/user (e.g., shopping cart merge).

## Consensus algorithms
- **Paxos:** the original proof that consensus is achievable; notoriously hard to implement correctly (Multi-Paxos for practical use).
- **Raft:** designed for understandability — leader election + log replication + safety via terms and commit index. Know the three roles (leader/follower/candidate), how leader election works (randomized timeouts), and how log replication/commit works (majority ack). This is what most interviewers actually want you to be able to explain.
- **ZAB (Zookeeper Atomic Broadcast):** similar goals to Raft, powers ZooKeeper.
- Used for: leader election, distributed locks, config/metadata stores (etcd, Consul, ZooKeeper), strongly consistent replicated logs.

## Quorums
`W + R > N` guarantees read-after-write consistency (you always overlap with at least one node that has the latest write). Common configs: N=3, W=2, R=2 (tolerates 1 node down, strong-ish consistency); N=3, W=1, R=1 (fast, eventually consistent).

## Common follow-ups
- "How does Raft handle a network partition with two sub-majorities?" → the minority side can't elect a leader (no majority), so it stops accepting writes; the majority side continues. Prevents split-brain.
- "What happens if the leader fails mid-write in single-leader replication?" → depends on sync vs async; discuss failover process (new leader election, potential data loss window, fencing tokens to prevent old leader from writing after failover).
- "How would you detect and handle split-brain?" → fencing tokens, quorum requirement for writes, STONITH-style node fencing.
