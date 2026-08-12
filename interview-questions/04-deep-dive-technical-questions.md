# Deep-Dive Technical Questions ("Explain how X works")

Rapid-fire depth checks, often asked as follow-ups mid-design or as standalone rounds. The bar for senior infra is being able to go *below* the "what" into the "how" and "why" — practice explaining each of these out loud in under 3 minutes, then drill into the harder follow-up.

## Distributed systems / consensus
- Explain how Raft leader election works. What prevents two leaders from being elected in the same term?
  - *Follow-up:* what happens on a network partition that splits the cluster into two groups of unequal size?
- Explain how Raft log replication and commit index work. What happens if a follower is behind by many entries?
- What is a quorum, and why does `W + R > N` give you read-after-write consistency?
- What's the difference between linearizability and serializability? (Different axes — one's about real-time recency of a single object, the other's about transaction isolation across multiple objects.)
- Explain vector clocks and why they can detect concurrency that Lamport timestamps can't.
- What is a fencing token and what specific race condition does it prevent?

## Storage
- Explain how an LSM-tree handles a write, and how a read has to check multiple places.
- What is compaction in an LSM-tree, and why does it matter for read amplification and write amplification?
- Explain how a bloom filter works and why it can have false positives but never false negatives.
- Walk through what happens end-to-end when you `INSERT` a row in a B-tree-based relational database with a WAL.
- Explain MVCC and how it lets readers avoid blocking on writers.
- What's the difference between optimistic and pessimistic concurrency control, and when would you choose each?
- How does consistent hashing work, and why do virtual nodes matter?

## Caching
- Walk through a cache stampede scenario and two different ways to prevent it.
- Explain the difference between write-through, write-behind, and write-around caching, with a scenario where each is the right choice.
- How would you keep a distributed cache consistent with a database under concurrent writes, and what's the residual risk?

## Networking
- Walk through a TCP handshake and explain what a SYN flood attacks.
- Explain the difference between TCP and UDP and why QUIC (HTTP/3) is built on UDP instead of TCP.
- Walk through a TLS 1.3 handshake at a high level. Where would you terminate TLS in a typical cloud architecture and why?
- Explain head-of-line blocking, and how HTTP/2 vs HTTP/3 each handle it differently.
- What happens, step by step, from typing a URL to the browser rendering the page? (Classic — but for infra, focus on DNS resolution, connection setup, and load balancing hops, not rendering.)
- Explain how a circuit breaker works — states, transitions, and why "half-open" exists.

## Messaging
- Explain exactly-once vs at-least-once delivery, and why true end-to-end exactly-once is effectively impossible without idempotent consumers.
- How does Kafka guarantee ordering, and what's the scope of that guarantee (partition, not topic)?
- Explain consumer group rebalancing in Kafka — what triggers it, and what's disruptive about it at scale.
- How would you design a dead-letter queue, and what's your strategy for reprocessing DLQ messages?

## Containers / orchestration
- Explain how Kubernetes decides where to schedule a pod.
- What's the difference between a liveness probe and a readiness probe, and what goes wrong if you configure them identically?
- Explain requests vs limits in Kubernetes and what happens when a pod exceeds each.
- What are Linux namespaces and cgroups, and which one is responsible for which part of container isolation?
- Explain what a Kubernetes controller/reconciliation loop is and why the model is "level-triggered."

## Observability
- Why can't you just always log everything at high cardinality? What's the actual cost model?
- Explain the difference between histogram and summary metric types, and why histograms aggregate better across instances.
- How does distributed trace context propagate across an async message queue boundary?
- Explain multi-window multi-burn-rate SLO alerting and why a single static threshold isn't enough.

## Reliability
- Walk through exactly how a cascading failure develops from one slow dependency, step by step.
- Explain the difference between RPO and RTO with a concrete example of each driving a design decision.
- What's the difference between load shedding and backpressure, and when do you need both?
- Explain why naive retries can make an outage worse, and the specific mechanisms (backoff, jitter, caps, circuit breakers) that fix it.

## Security-adjacent (often folded into infra loops)
- Explain mTLS and why it's the standard for service-to-service auth in a mesh.
- How would you design short-lived credential issuance for services (vs long-lived static secrets), and why is that safer?
- What's the "bootstrapping problem" for secrets — how does a freshly started service authenticate to the secrets store in the first place?

## How to use this list
Don't just read the answers in your head — say them out loud, unprompted, in under 3 minutes each, then ask yourself the natural follow-up an interviewer would ask and answer that too. If you can't generate the follow-up yourself, you don't know the topic deeply enough yet — go back to the matching fundamentals doc.
