# Design a Distributed Lock Service (like ZooKeeper/etcd's lock primitive)

## Clarifying questions to ask first
- Do clients need reentrant locks (same client can re-acquire a lock it already holds), or is non-reentrant sufficient?
- What should happen to waiters when the lock holder dies — how quickly must the lock be reclaimed?
- Is this for coarse-grained locks (leader election, rare) or fine-grained/high-throughput locking?

## Requirements
### Functional
- `acquire(lock_name)`, `release(lock_name)`, with a lease/TTL so locks aren't held forever by a dead client.
- Notify or unblock waiters when a lock becomes available.

### Non-functional
- Must never allow two clients to simultaneously believe they hold the same lock (safety is non-negotiable here — this is the whole point of the service).
- Should recover automatically when a lock holder crashes, without requiring manual intervention.
- Fairness: waiters shouldn't be starved indefinitely by new arrivals.

## High-level architecture
Clients → lock service API → underlying consensus-replicated store (built on Raft, like etcd/ZooKeeper) holding lock state (owner, lease expiry, waiter queue) → watch/notification mechanism to wake up waiters.

## Deep dives

### Consensus underneath (Raft)
- The lock service's state (who holds which lock, current lease expiry) must be replicated consistently across nodes so that a leader failure doesn't lose or duplicate lock state. Raft gives you a replicated log with a single elected leader handling all writes, replicated to a majority before being considered committed — this is what makes "only one client can hold the lock" enforceable even as the lock service's own nodes fail.
- Because Raft requires a majority (quorum) to commit, the lock service itself keeps working through a minority-side failure, but a client on the minority side of a network partition can no longer acquire or renew locks — which is the correct, safe behavior (better to be unavailable than to allow two simultaneous "leaders").

### Lease/TTL model
- A lock is granted with a TTL; the holder must periodically renew (heartbeat) before it expires, or the lock is automatically released for others to acquire. This is what makes the system self-healing when a holder crashes — no manual cleanup needed, the lease just expires.
- TTL is a real tradeoff: too short and a holder doing legitimate slow work (or hit by a GC pause) loses the lock and someone else acquires it while the first thread thinks it's still safe; too long and a genuinely crashed holder blocks everyone else for that whole duration.

### Fencing tokens (the answer interviewers are fishing for)
- The TTL/lease design has an unavoidable race: a client can be paused (GC pause, slow disk, network delay) for long enough that its lease expires and gets granted to someone else, then wake up still believing it holds the lock and perform its operation — now two clients act as if they hold the lock simultaneously.
- **Fix**: every time the lock is granted, issue a monotonically increasing **fencing token** (e.g., 1, 2, 3, ...) along with it. The client includes this token with every operation on the protected resource. The resource itself (e.g., a storage system) rejects any operation with a token lower than the highest one it's already seen. This turns "the lock service says who *should* be able to write" into "the protected resource enforces who *is actually allowed* to write," closing the race — this is the detail that separates a correct answer from a naively "good enough" one.

### Reentrant vs non-reentrant locks
- Non-reentrant is simpler and safer by default (a client can't accidentally deadlock itself, but also can't recursively acquire the same lock across nested calls without explicit handling).
- Reentrant requires tracking an owner identity plus a hold count, and must be scoped correctly (per-thread vs per-process identity) or you introduce subtle bugs where two different threads on the same client accidentally share reentrancy.

### Fairness (FIFO queue vs thundering herd)
- Naive "notify everyone on release" causes a thundering herd — every waiter wakes up, races to acquire, all but one fail and go back to waiting, wasting resources at scale.
- Better: maintain an explicit FIFO queue of waiters (this is literally how ZooKeeper recipes implement fair locks — each waiter creates a sequential ephemeral node and only watches the node immediately ahead of it in the queue, so only one waiter is ever woken on release).

## Key tradeoffs
- Short TTL (fast recovery from a dead holder, but risk of premature expiry under transient slowness) vs long TTL (safer against false expiry, slower recovery from real crashes).
- Fencing tokens add a small amount of integration work (the protected resource must check them) in exchange for closing a real correctness gap that lease-only designs have.

## Failure modes
- Lock service leader dies: Raft elects a new leader from the remaining quorum; in-flight lock state was already replicated to a majority so no committed lock grant is lost, but clients see a brief unavailability during the election.
- Network partition isolates a lock holder from the service: its lease will expire (it can't renew), the lock gets reassigned; if the isolated client later reconnects and tries to act as if it still holds the lock, fencing tokens are what stop it from corrupting shared state.

## Likely follow-ups
- "Walk me through the exact race that fencing tokens prevent, step by step." → client A acquires lock with token 5, pauses (GC) past its TTL; service expires A's lease and grants the lock to client B with token 6; B writes to the resource, resource records "last token seen: 6"; A wakes up and tries to write with token 5; resource rejects it because 5 < 6.
- "How would you build a read-write lock on top of this primitive?" → separate lock state for readers (count of concurrent holders, no exclusivity) vs a writer (fully exclusive, and must wait for the reader count to hit zero) — same lease/fencing mechanics apply to the writer slot.
