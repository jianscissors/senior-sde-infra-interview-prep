# Design a Centralized Log Aggregation System (like an internal ELK/Splunk)

## Clarifying questions to ask first
- What's the expected log volume (events/sec, bytes/day) at peak, and how bursty is it (e.g., an incident causing a 100x log spike)?
- Do consumers need full-text search, structured field queries, or both?
- What retention/tiering is required — how long must logs stay searchable at full detail vs archived?
- Multi-tenant (many teams sharing this system) — do we need isolation guarantees?

## Requirements
### Functional
- Collect logs from every host/service, make them searchable by text and structured fields, support alerting on log patterns.

### Non-functional
- Must not drop logs silently under normal load, and must degrade predictably (not catastrophically) under burst load — logs are often most valuable exactly during the incident that's also causing the volume spike.
- Query latency acceptable for interactive debugging (seconds, not minutes) on recent data.
- One noisy tenant's log volume shouldn't degrade search/ingestion for everyone else.

## High-level architecture
Host/service agent (tails log files or receives structured events) → buffer/queue (absorbs bursts, decouples producers from the indexing tier) → indexer (parses, extracts fields, writes to search-optimized storage) → tiered storage (hot/warm/cold) → query/search API + dashboards + alerting on log-derived signals.

## Deep dives

### Ingestion pipeline: agents → buffer/queue → indexer
- Agents (e.g., a Fluentd/Vector/Filebeat-style sidecar) tail local log output and forward it, doing light local parsing/enrichment (adding host, service, and environment tags) before shipping — cheaper to do this once at the edge than repeatedly downstream.
- A durable buffer/queue (Kafka is the common real-world choice) sits between agents and indexers specifically to absorb bursts: indexing is the expensive, rate-limited step, while ingestion into a queue is cheap and can absorb a spike that the indexer couldn't keep up with in real time, smoothing it out over the following minutes.

### Handling burst volume without dropping logs
- **Backpressure**: if the queue itself starts filling up faster than indexers can drain it, agents should slow down or buffer locally rather than the queue silently dropping messages — but local buffering is itself bounded (disk/memory), so this only buys time, not an unlimited safety margin.
- **Sampling under extreme load**: as a last resort during a true overload (e.g., a bug causing log-volume amplification), sample down verbose/low-value log levels (debug/info) while always preserving error/warn logs at full fidelity — losing some debug noise is an acceptable tradeoff, losing the error logs that would explain an ongoing incident is not.
- Both mechanisms should be visible (metrics on queue depth, drop/sample rate) so an operator can tell the difference between "system healthy, nothing to see" and "system quietly discarding data."

### Indexing/query tradeoffs
- Full-text inverted indexes (Elasticsearch-style) give flexible free-text search but are expensive to build and store — indexing cost scales with ingestion volume, not just query volume, which is what makes log systems expensive at scale.
- Structured field extraction (parsing known fields like `status_code`, `latency_ms` out of each log line at ingest time) enables much cheaper, much faster queries for the common case ("show me all 5xx responses in the last hour") than fall-back full-text scanning — encourage structured logging from services specifically to make this the common case rather than the exception.

### Retention tiers (hot/warm/cold)
- **Hot**: recent data (e.g., last 3-7 days), fully indexed, fast interactive queries — this tier is the most expensive per byte and should be sized to the realistic "someone is actively debugging something" window.
- **Warm**: older data, indexed but on cheaper/slower storage, acceptable to have higher query latency.
- **Cold/archive**: compressed, unindexed or minimally indexed blob storage (e.g., S3), for compliance/rare historical lookups — a query here might mean a batch job, not an interactive search, and that's an acceptable tradeoff given how rarely it's needed.
- Automate the rollover between tiers on age, and make retention policy configurable per log type/tenant, since compliance-relevant logs and debug-only logs often have very different real requirements.

### Multi-tenant isolation
- Without isolation, one team's misbehaving service (e.g., accidentally logging at extreme volume) can starve ingestion/indexing capacity for every other team. Enforce per-tenant ingestion quotas and, ideally, separate resource pools (or at least separate queue partitions) for indexing so one tenant's burst degrades only their own query latency, not everyone's.

## Key tradeoffs
- Full-text indexing (flexible, expensive) vs structured field extraction (cheaper/faster, requires disciplined logging practices upstream).
- Buffering (protects against loss, but only up to a bounded capacity) vs sampling (bounds cost/load, but is an explicit, intentional loss of data — must be scoped to only the lowest-value log levels).

## Failure modes
- Indexing tier falls behind during an incident-driven log spike: buffer/queue absorbs it temporarily; if the spike outlasts the buffer's capacity, sampling of low-priority levels kicks in before anything drops errors/warnings.
- A single tenant's runaway logging: per-tenant quota kicks in, isolating the impact to that tenant's own ingestion/query experience rather than degrading the shared system.

## Likely follow-ups
- "During a major incident, log volume 50x's for 10 minutes — walk me through what happens end to end." → agents keep shipping into the queue (queue absorbs the burst); if sustained past buffer capacity, debug/info sampling kicks in while errors/warnings are preserved; indexers drain the backlog over the following minutes; on-call sees a "logs delayed" indicator rather than silent gaps.
- "How would you support both 'grep-style' ad hoc search and structured dashboards on the same data?" → structured field extraction at ingest feeds both a full-text index (for ad hoc search) and a columnar/aggregation-friendly store (for dashboard-style aggregate queries), rather than forcing one query engine to serve both patterns well.
