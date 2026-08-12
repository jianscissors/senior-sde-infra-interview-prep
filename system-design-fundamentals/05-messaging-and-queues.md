# Messaging, Queues & Event Streaming

## Why it comes up
Async communication is central to infra design — decoupling services, absorbing load spikes, enabling event-driven architectures. Interviewers probe whether you understand delivery guarantees and ordering, not just "put a queue between them."

## Why use a queue/message broker at all
- Decouple producers and consumers in time and rate (buffer spiky load, smooth it out for downstream).
- Enable async workflows (fire-and-forget, background jobs).
- Fan-out to multiple consumers (pub/sub).
- Improve resilience — a consumer outage doesn't lose messages, they just back up.

## Queue vs pub/sub vs log
- **Point-to-point queue (e.g., SQS, RabbitMQ queues):** each message consumed by exactly one consumer (within a consumer group). Good for work distribution/task queues.
- **Pub/sub (e.g., SNS, Redis Pub/Sub):** each message delivered to all subscribers. Good for broadcast/notifications. Usually no replay.
- **Log-based streaming (e.g., Kafka, Kinesis):** durable, ordered, replayable append-only log partitioned by key; multiple independent consumer groups each read the full stream at their own pace. Good for event sourcing, stream processing, multiple downstream consumers needing the same data.

## Delivery guarantees (know precisely — this is a favorite interview probe)
- **At-most-once:** message may be lost, never redelivered. Fire-and-forget, no acks.
- **At-least-once:** message is retried until acked, so it may be delivered more than once. Requires **idempotent** consumers. This is the most common real-world default.
- **Exactly-once:** hardest to achieve; usually means at-least-once delivery + idempotent processing (dedup by message ID), or transactional producer/consumer support (e.g., Kafka's idempotent producer + transactions). True end-to-end exactly-once across arbitrary systems is effectively impossible without cooperation from both ends — say this explicitly if asked.

## Ordering
- Global ordering across a whole topic is expensive and usually unnecessary.
- Kafka-style: ordering guaranteed only **within a partition**. Choose your partition key carefully (e.g., user ID, entity ID) so events for the same entity stay ordered.
- Out-of-order handling: sequence numbers/versions on messages, consumer-side reordering buffers, or design the consumer to be commutative (order-independent) when possible.

## Idempotency (comes up constantly, not just for messaging)
- Deduplicate using a unique message/request ID stored in the consumer's processed-set (with TTL, or a dedup table with a unique constraint).
- Design operations to be naturally idempotent (e.g., "set balance to X" instead of "add X"; upserts instead of inserts).
- Idempotency keys on the producer side for retried requests (standard pattern for payment APIs).

## Backpressure & flow control
- Bounded queues + rejecting/shedding load when full (better than unbounded growth → OOM).
- Consumer-driven pull (Kafka-style) naturally provides backpressure — consumer reads at its own pace.
- Push-based systems need explicit credit/windowing (like TCP flow control) to avoid overwhelming slow consumers.
- Dead-letter queues (DLQ) for messages that repeatedly fail processing, so one poison message doesn't block the whole queue.

## Kafka architecture (common deep-dive)
- Topics split into partitions, each partition is an ordered append-only log, replicated across brokers (one leader, N followers per partition).
- Producers write to a partition (by key hash or round robin); consumers in a consumer group split partitions among themselves (each partition consumed by one consumer in the group at a time).
- Offsets track consumer progress per partition, committed to allow resume after restart.
- Retention is time/size-based, not consumption-based — multiple consumer groups can replay independently.
- `acks=all` + `min.insync.replicas` control the durability/latency tradeoff.

## Common follow-ups
- "How do you prevent duplicate processing when a consumer crashes after processing but before committing the offset?" → this is exactly why consumers must be idempotent; at-least-once + idempotent handler is the standard answer.
- "How would you handle a slow consumer without blocking the whole system?" → per-consumer-group independence (log-based systems), backpressure, or scale out consumers/partitions.
- "Design a system to guarantee ordered delivery per user at massive scale" → partition by user ID, single consumer per partition per group, accept that total throughput for one user is bounded by one partition's throughput.
