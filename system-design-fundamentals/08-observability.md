# Observability: Metrics, Logging, Tracing, Alerting

## Why it comes up
Infra interviews often include "design a monitoring/metrics system" directly, and every other design should mention observability as part of the answer — senior candidates are expected to design for operability, not just correctness.

## The three (four) pillars
- **Metrics:** numeric time-series (counters, gauges, histograms) — cheap to store/query at scale, good for dashboards/alerting/trends, but low cardinality (can't ask "show me this one user's request" from metrics alone).
- **Logs:** structured/unstructured event records — high detail, high cardinality, expensive to store/query at scale, good for debugging a specific incident.
- **Traces:** a request's path across services with timing per hop (spans) — essential for finding where latency/errors originate in a distributed call graph.
- **Events (sometimes called the 4th pillar):** discrete significant occurrences (deploys, config changes, alerts firing) — correlating these with metrics/logs/traces is often what actually finds root cause ("latency spiked right after the deploy at 14:32").

## Metrics pipeline design
- **Push vs pull:** Prometheus-style pull (server scrapes `/metrics` endpoints periodically) is simple, self-limiting (scraper controls load), needs service discovery to know what to scrape. Push (StatsD, CloudWatch-style) is needed for short-lived jobs (can't be scraped in time) and works better through firewalls/NAT.
- **Aggregation:** raw high-cardinality metrics get rolled up (e.g., per-minute → per-hour after N days) to bound storage — a time-series DB (Prometheus, InfluxDB, M3, Mimir) is purpose-built for this.
- **Cardinality control:** labels/tags multiply storage cost combinatorially (a metric with `user_id` as a label can explode into millions of series) — this is the #1 real-world metrics-system incident cause; keep high-cardinality dimensions in logs/traces, not metrics.
- **Histograms vs summaries:** histograms (bucketed counts) can be aggregated across nodes server-side and support arbitrary percentile queries later; summaries (pre-computed quantiles on the client) can't be meaningfully aggregated across instances — prefer histograms for anything you'll aggregate.

## Logging pipeline design
- Structured logging (JSON, key-value) over free text — enables querying/filtering at scale.
- Pipeline shape: app → local agent (Fluentd/Fluent Bit/Vector) → buffer/queue (Kafka) → indexing/storage (Elasticsearch, ClickHouse, cheaper object storage for cold logs) → query layer.
- Sampling: at high volume, log 100% of errors but sample a percentage of normal/info logs — bound cost while keeping signal.
- Correlation IDs: propagate a request/trace ID through every log line across services so you can pull the full story of one request.

## Distributed tracing
- **Span:** a single unit of work (one hop/operation) with start/end time and metadata; **trace:** a tree/DAG of spans for one end-to-end request.
- Context propagation: trace ID + span ID passed via headers (W3C Trace Context standard) across every service hop, including through async boundaries (queues) — this is the hard/easy-to-get-wrong part in practice.
- Sampling strategies: head-based (decide to sample at the start, cheap but may miss rare slow/error traces), tail-based (buffer and decide after seeing the full trace, catches interesting traces but needs a buffering collector — more infra).
- Tools: Jaeger, Zipkin, OpenTelemetry (the emerging standard SDK/collector for all three pillars, vendor-neutral).

## Alerting
- Alert on **symptoms** (user-facing: error rate, latency, saturation) not just causes (CPU high) — a cause without a symptom often doesn't need to page anyone at 3am.
- **SLO-based alerting / error budgets:** define an SLO (e.g., 99.9% of requests succeed under 300ms), alert on burn rate against the error budget rather than a fixed static threshold — catches both fast catastrophic failures and slow persistent degradation, with fewer false pages than naive thresholds.
- Multi-window, multi-burn-rate alerts (Google SRE workbook pattern): a fast-burn short window catches sudden outages, a slow-burn long window catches persistent low-grade issues, without either one flooding on-call with noise.
- Deduplication/grouping and routing (PagerDuty/Opsgenie-style) so one root cause doesn't page 50 separate alerts.

## Common follow-ups
- "How do you keep a metrics system from falling over under high cardinality?" → cap/reject high-cardinality labels at ingestion, push high-cardinality data to logs/traces instead, pre-aggregate.
- "Design a system to alert on p99 latency across a fleet of 10k hosts." → histogram-based metrics aggregated centrally (not per-host summaries), SLO burn-rate alerting, dedup/routing layer.
- "How do you trace a request that goes through an async queue?" → propagate trace context in the message envelope/headers, the consumer starts a new span linked (not just parented) to the producer's span since it's async.
