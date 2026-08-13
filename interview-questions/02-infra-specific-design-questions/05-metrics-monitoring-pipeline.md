# Design a Metrics/Monitoring Pipeline for 100K Hosts (like Prometheus at scale, or Datadog's ingestion path)

## Clarifying questions to ask first
- Pull-based (central system scrapes each host) or push-based (hosts ship metrics out)?
- What's the metric cardinality expectation — how many unique label combinations per metric?
- Query patterns: mostly dashboards (recent data, coarse resolution) or also alerting (needs low-latency recent data)?
- Retention requirements — how long does raw-resolution data need to be queryable?

## Requirements
### Functional
- Ingest time-series metrics (counters, gauges, histograms) from 100K hosts.
- Support dashboard queries and real-time alerting on top of the same data.

### Non-functional
- Ingestion must keep up with 100K hosts each emitting on the order of hundreds of metrics every 10-15s — that's on the order of millions of data points per minute.
- Alerting needs low end-to-end latency (seconds, not minutes) from a metric being emitted to an alert firing.
- Storage cost must stay bounded as retention and host count grow — raw-resolution-forever doesn't scale.

## High-level architecture
Host agent (collects local metrics) → [pull: central scraper fetches; or push: agent ships to a local aggregator] → ingestion/aggregation tier → time-series storage (with compaction/downsampling) → query layer (for dashboards) + alerting rule evaluator running continuously against the same store.

## Deep dives

### Pull vs push
- **Pull (Prometheus model)**: central servers scrape each host's `/metrics` endpoint on an interval. Advantages: the scraper naturally knows if a host is unreachable (a failed scrape *is* a signal), and hosts don't need to know where to send data, simplifying host-side config. Downside: doesn't scale to 100K hosts from one scraper — needs federation/sharding of scrapers by host group.
- **Push (Datadog/statsd-style model)**: hosts actively ship metrics to a local agent, which forwards them onward. Scales more naturally horizontally (no single scraper bottleneck) and works better across networks where hosts aren't directly reachable (behind NAT, ephemeral containers). Downside: harder to distinguish "host is fine but stopped reporting" from "host is dead," and needs backpressure handling if the ingestion tier falls behind.
- At 100K-host scale, most real systems end up push-based (or pull-with-federation, which is functionally similar) specifically because a single pull point can't reach that many hosts fast enough at a useful interval.

### Cardinality control
- Every unique combination of metric name + label values creates a new time series. A seemingly innocent label like `user_id` or `request_id` on a metric can explode cardinality from thousands of series to tens of millions, blowing up both ingestion cost and query latency (this is the single most common real-world cause of a monitoring system falling over).
- Mitigation: enforce label cardinality limits at ingestion (reject or drop high-cardinality labels), push genuinely high-cardinality data (like per-request traces) to a different system (tracing/logging) instead of metrics, and document/lint against unbounded labels in client libraries before they ever reach production.

### Local aggregation before shipping
- Rather than shipping every raw data point from every host, aggregate locally first (e.g., a local agent pre-aggregates a counter over a 10s window before sending one point instead of many). This cuts network volume and central ingestion load by an order of magnitude, at the cost of losing sub-window resolution.
- This is also where histogram pre-bucketing happens — shipping pre-computed bucket counts rather than raw observations keeps the payload size bounded regardless of request volume on that host.

### Time-series storage and compaction
- Recent data is written and queried at full resolution; older data gets **downsampled** (e.g., roll 10s-resolution data up to 5-minute resolution after 24 hours, and to 1-hour resolution after 30 days) to keep long-term storage bounded while still supporting "what did this look like 6 months ago" trend queries.
- Storage engines for this (Prometheus's own TSDB, or systems like Gorilla/M3DB) use compression tuned for time series specifically — delta-of-delta encoding for timestamps, XOR encoding for slowly-changing float values — because naive storage of raw floats+timestamps is extremely wasteful for this access pattern.

### Query layer
- Needs to serve both ad-hoc dashboard queries (can tolerate slightly higher latency) and continuously-running alert rule evaluations (need to run frequently and reliably, since a missed or delayed alert evaluation is effectively a missed incident detection). Many systems run the alerting evaluator as a distinct, prioritized path rather than sharing a query queue with interactive dashboard traffic.

## Key tradeoffs
- Pull's operational simplicity and implicit liveness signal vs push's better horizontal scalability at very large host counts.
- Full-resolution-forever (expensive, simple) vs downsampling (cheap, but you permanently lose fine-grained historical detail past the downsampling horizon).

## Failure modes
- Ingestion tier falls behind: must decide between backpressure (agents buffer and retry, risking data loss if the buffer overflows) and load shedding (drop excess data now, accept gaps, to protect the pipeline's overall health) — see the reliability fundamentals doc for this same tradeoff in general.
- A single host or group goes silent: the system must distinguish "expected, host was decommissioned" from "unexpected, host or its agent crashed" — usually via an explicit deregistration signal or an alert on "stopped reporting" as its own condition.

## Likely follow-ups
- "A new deploy introduced a label with unbounded cardinality and ingestion is falling over — what do you do right now, and how do you prevent it next time?" → immediate: drop/block the offending label at the ingestion edge; longer-term: cardinality limits enforced in the client library and caught in CI/review, not just at runtime.
- "How would you reduce alerting false positives at this scale without missing real incidents?" → multi-window multi-burn-rate alerting (see the observability fundamentals doc) instead of a single static threshold.
