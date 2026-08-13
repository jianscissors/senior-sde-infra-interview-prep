# Design a Distributed Tracing System's Ingestion Pipeline (like a simplified Jaeger backend)

## Clarifying questions to ask first
- What's the expected span volume (spans/sec) across the whole fleet, and how many services participate in a typical trace?
- Sampling: head-based (decided at the start of a trace, by the originating service) or tail-based (decided after seeing the whole trace, so you can sample based on outcome — e.g., always keep error traces)?
- What's the required query pattern: lookup by trace ID (point lookup) primarily, or also search/filter by service+time-range+tags?
- Retention requirement for traces?

## Requirements
### Functional
- Ingest spans emitted by instrumented services, each tagged with a trace ID, span ID, parent span ID, timestamps, and metadata.
- Assemble spans belonging to the same trace ID into a coherent trace, and make it queryable (by trace ID, and by service/time-range/tag filters).

### Non-functional
- Must ingest at very high volume without materially adding latency to the instrumented services themselves (span emission must be fire-and-forget from the app's perspective).
- Storage must be optimized for the actual query patterns (trace-ID lookup and time-range queries), not for OLTP-style arbitrary queries.

## High-level architecture
1. **SDK/agent in each service**: generates spans locally, buffers them, and ships them asynchronously (never blocking the request path) to a local agent or directly to a collector.
2. **Collector tier**: receives spans from many services, does validation/enrichment, applies sampling decisions (if not already decided at the SDK), and forwards to storage.
3. **Storage**: optimized for trace-ID point lookups and time-range scans — commonly a wide-column or time-series-friendly store, often with a search index (e.g., Elasticsearch) layered on top for tag-based queries.
4. **Query service**: serves the UI/API, reassembling all spans for a trace ID into the parent-child tree.

## Deep dives

### Span ingestion at high volume
- Every span emission from the instrumented service must be async and non-blocking — buffered locally (in-process or via a local sidecar agent) and flushed in batches, so a slow or unavailable tracing backend never adds latency to, or causes failures in, the actual user-facing request path. This is the single most important property of the SDK design.

### Sampling strategy: head-based vs tail-based
- **Head-based sampling**: the originating service decides, at trace start, whether to sample this trace (e.g., sample 1% of all traces, or 100% of a specific high-value endpoint), and that decision propagates to every downstream service via the trace context. Simple, cheap (no buffering needed), but you can't sample based on the *outcome* of the trace — you decide before you know if it errored or was slow.
- **Tail-based sampling**: every span for a trace is buffered (at the collector tier) until the full trace completes, and *then* a sampling decision is made — commonly "always keep traces with an error or with latency above a threshold, sample everything else at a low rate." This gives much higher-value sampled data (you actually keep the interesting traces) but requires buffering all spans for all in-flight traces at the collector until each trace is deemed complete, which is significantly more expensive in memory and requires routing all spans for the same trace ID to the same collector instance.

### Trace assembly from out-of-order spans
- Spans for a single trace arrive from many different services, over different network paths, and are not guaranteed to arrive in causal order — a child span can arrive before its parent. The collector/storage layer needs a **buffering window** per trace ID (wait N seconds after the first span for a trace ID before considering it "complete enough" to index/serve, or serve partial traces that get updated as more spans trickle in) rather than assuming spans arrive in order.
- For tail-based sampling specifically, this requires all spans belonging to one trace ID to be routed to (or eventually reconciled at) the same collector instance — typically by consistently hashing on trace ID when load-balancing from agents to collectors.

### Storage model
- Point lookups by trace ID need an index keyed directly on trace ID. Time-range + service/tag filter queries (e.g., "show me slow traces for service X in the last hour") benefit from a separate search index (inverted index on tags/service name, time-bucketed) rather than trying to serve both access patterns from one data structure — this is why real systems like Jaeger commonly pair a primary span store with Elasticsearch or Cassandra-plus-an-index for the search path.

## Key tradeoffs
- Tail-based sampling gives much more useful retained data (you keep the traces that actually matter) but costs significantly more in buffering memory and collector-side complexity than head-based sampling.
- High retention/high sampling rate gives better debugging coverage but costs proportionally more storage — most systems sample aggressively for the bulk of "boring" traffic and keep 100% of errors/outliers.

## Failure modes
- Collector tier falls behind ingest volume: spans queue up and, if buffers fill, get dropped — better to drop spans under overload (graceful degradation of tracing fidelity) than to apply backpressure that could ever propagate back to the instrumented service's request path.
- A trace's spans get split across collectors inconsistently (e.g., under tail-based sampling without consistent hashing): partial/incomplete traces get incorrectly sampled out or shown incomplete — this is why consistent routing by trace ID matters.

## Likely follow-ups
- "How do you keep sampling from hiding a real production issue that affects only a small percentage of requests?" → always retain 100% of error/high-latency traces (tail-based, or a head-based override for known-risky endpoints); low-value uniform sampling should only apply to routine, healthy traffic.
- "How does trace context propagate across an async message queue boundary (not a direct synchronous call)?" → the trace context (trace ID, parent span ID) must be serialized into the message's metadata/headers by the producer and read back out by the consumer to continue the same trace — without this, tracing breaks at every async boundary and you get disconnected trace fragments instead of one end-to-end trace.
