# Infra-Specific System Design Questions

These are the questions that actually differentiate an infra-track loop from a generic backend one — you're often designing the platform/tooling other engineers build on top of, not a consumer-facing product. Spend the most practice time here. For each, the "focus areas" are what interviewers are usually listening for.

Each question below has a full detailed write-up (clarifying questions, requirements, architecture, deep dives, tradeoffs, failure modes, likely follow-ups) in `02-infra-specific-design-questions/`. The bullets here are the quick-reference summary — click through before a practice session, not during one.

1. **[Design a distributed cache](02-infra-specific-design-questions/01-distributed-cache.md)** (like a simplified Redis Cluster/Memcached)
   - Focus: consistent hashing for sharding, replication for durability/availability, eviction policy, handling hot keys, client protocol (thin client vs proxy layer like Twemproxy/Envoy), failover when a shard's primary dies.

2. **[Design a rate limiter service used by hundreds of internal microservices](02-infra-specific-design-questions/02-shared-rate-limiter-service.md)**
   - Focus: as a *shared service* not embedded logic — API design, algorithm choice (sliding window counter is a common sweet spot), storage for counters (Redis with atomic INCR+EXPIRE, or local approximate counting with periodic sync for lower latency), multi-region consistency (usually approximate/eventually consistent limits are acceptable), fail-open vs fail-closed when the limiter itself is down.

3. **[Design a distributed job scheduler](02-infra-specific-design-questions/03-distributed-job-scheduler.md)** (like a simplified Kubernetes scheduler, or Airflow/cron-at-scale)
   - Focus: job queue and worker pool model, leader election for the scheduler itself (avoid double-scheduling), handling worker failure mid-job (lease/heartbeat + idempotent re-execution), priority/fairness across tenants, dependency graphs (DAG execution) if relevant, backfill/catch-up semantics.

4. **[Design a distributed lock service](02-infra-specific-design-questions/04-distributed-lock-service.md)** (like ZooKeeper/etcd's lock primitive)
   - Focus: consensus underneath (Raft), lease/TTL model, fencing tokens (this is the answer interviewers are fishing for), reentrant vs non-reentrant locks, fairness (FIFO queue of waiters vs thundering herd on release).

5. **[Design a metrics/monitoring pipeline for 100k hosts](02-infra-specific-design-questions/05-metrics-monitoring-pipeline.md)** (like Prometheus at scale, or Datadog's ingestion path)
   - Focus: pull vs push, cardinality control, local aggregation before shipping (reduce network/storage), time-series storage/compaction strategy, query layer for dashboards, alerting on top (see observability doc for SLO burn-rate alerting).

6. **[Design a centralized log aggregation system](02-infra-specific-design-questions/06-centralized-log-aggregation.md)** (like an internal ELK/Splunk)
   - Focus: ingestion pipeline (agents → buffer/queue → indexer), handling burst volume without dropping logs (backpressure, buffering, sampling under extreme load), indexing/query tradeoffs, retention tiers (hot/warm/cold storage), multi-tenant isolation so one noisy team doesn't starve others.

7. **[Design an object storage system](02-infra-specific-design-questions/07-object-storage-system.md)** (like a simplified S3)
   - Focus: metadata service (key→location) vs data plane separation, durability (erasure coding vs replication tradeoffs), consistency model (strong read-after-write), multipart upload for large objects, hot object handling (a viral file), lifecycle/tiering.

8. **[Design a distributed file system](02-infra-specific-design-questions/08-distributed-file-system.md)** (like HDFS/GFS)
   - Focus: single metadata server (NameNode) vs distributed metadata, block-based storage with replication, write-once-read-many optimization, handling the metadata-server SPOF/scale ceiling (this is exactly why later systems moved to federated/distributed metadata).

9. **[Design a service discovery system](02-infra-specific-design-questions/09-service-discovery.md)** (like Consul/etcd-backed discovery, or client-side discovery for gRPC)
   - Focus: registration (self-register with heartbeat vs sidecar-managed), health-checking integration, propagation latency vs consistency tradeoff (how fast does a new/dead instance get reflected), client-side caching with fallback on registry unavailability, DNS-based vs API-based discovery.

10. **[Design a CI/CD pipeline system](02-infra-specific-design-questions/10-cicd-pipeline-system.md)** (like a simplified GitHub Actions/Jenkins/Buildkite at company scale)
    - Focus: build queue and worker fleet (isolated, often ephemeral containers/VMs per build for security), artifact caching (layer/dependency caching to speed builds), fan-out for parallel test sharding, secrets management for pipelines, multi-tenant fairness/priority across teams, deployment gating/approval workflows.

11. **[Design a container registry](02-infra-specific-design-questions/11-container-registry.md)** (like a simplified Docker Hub/ECR)
    - Focus: content-addressable storage (layers deduped by digest), manifest/layer separation, push/pull protocol, garbage collection of untagged/orphaned layers, replication across regions for pull latency, vulnerability scanning pipeline as an async hook.

12. **[Design a service mesh's control plane](02-infra-specific-design-questions/12-service-mesh-control-plane.md)** (like a simplified Istio control plane)
    - Focus: how config (routing rules, mTLS certs, retries/circuit-breaking policy) propagates to thousands of sidecar proxies without overwhelming the control plane, xDS-style incremental updates, certificate rotation at scale, blast radius if the control plane itself is degraded (data plane should keep working with last-known-good config — "control plane down" should not mean "data plane down").

13. **[Design a feature flag / experimentation platform](02-infra-specific-design-questions/13-feature-flag-experimentation-platform.md)**
    - Focus: low-latency flag evaluation (must not add meaningful request latency — usually means local SDK evaluation against a periodically-synced ruleset, not a network call per check), consistent bucketing (same user always gets the same variant, via hashing), propagation latency for flag changes, kill-switch requirements (must propagate fast, sub-second, when something needs to be turned off immediately).

14. **[Design a distributed message queue](02-infra-specific-design-questions/14-distributed-message-queue.md)** (like a simplified Kafka)
    - Focus: partitioning and ordering-within-partition, replication (leader/follower per partition), consumer group rebalancing, offset management, durability/`acks` tradeoffs, handling a broker failure without data loss or long unavailability.

15. **[Design a config management / dynamic configuration system](02-infra-specific-design-questions/15-config-management-dynamic-config.md)** (like a simplified internal config service backing thousands of services)
    - Focus: propagation speed vs consistency (usually eventual, fast propagation via push/watch is prioritized over strong consistency), versioning and rollback, validation/staged rollout of config changes (a bad global config push is a classic large-scale outage cause — mention canarying config changes just like code changes), local caching so services degrade gracefully if the config service is briefly unavailable.

16. **[Design an autoscaler](02-infra-specific-design-questions/16-autoscaler.md)** for a fleet of services
    - Focus: metrics-driven scaling decisions (CPU/queue depth/custom metrics), avoiding flapping (cooldowns, hysteresis), scale-out speed vs scale-in caution, predictive/scheduled scaling for known traffic patterns, interaction with load balancer health checks and connection draining during scale-in.

17. **[Design a secrets management system](02-infra-specific-design-questions/17-secrets-management-system.md)** (like a simplified Vault)
    - Focus: encryption at rest and in transit, access control (least-privilege policies per service identity), dynamic/short-lived credentials vs static secrets, audit logging of every access, rotation without downtime, bootstrapping problem (how does a service authenticate to the secrets store in the first place — often via a platform-provided identity like a cloud IAM role or a k8s service account token).

18. **[Design a global traffic management / multi-region failover system](02-infra-specific-design-questions/18-global-traffic-management-multi-region-failover.md)**
    - Focus: health-checking each region, DNS/GSLB-based routing with TTL tradeoffs, active-active vs active-passive, data replication lag's effect on failover safety (failing over to a region with stale data), automated vs human-gated failover for a system this risky.

19. **[Design a distributed tracing system's ingestion pipeline](02-infra-specific-design-questions/19-distributed-tracing-ingestion-pipeline.md)** (like a simplified Jaeger backend)
    - Focus: span ingestion at high volume, sampling strategy (head-based at the SDK vs tail-based needing a buffering collector), trace assembly (spans arrive out of order from different services — need a window to assemble), storage model optimized for trace-ID lookups and time-range queries.

20. **[Design a bin-packing / resource scheduler for a compute cluster](02-infra-specific-design-questions/20-bin-packing-resource-scheduler.md)** (the "how does the K8s scheduler actually decide" question, generalized)
    - Focus: bin-packing algorithm tradeoffs (best-fit vs spread for availability vs cost), handling resource fragmentation over time, priority/preemption for higher-priority workloads, multi-dimensional constraints (CPU, memory, GPU, disk, network), fairness across tenants sharing the cluster.

## Practice tips specific to this set
- These questions reward knowing *real systems* by name (Kafka, Raft, Kubernetes, S3) — cite the real design when relevant ("this is essentially what Kafka does with...") but always explain *why*, don't just namedrop.
- Interviewers will often push you toward a specific hard sub-problem (e.g., "ok now the leader dies mid-write, what happens") — practice going 2-3 levels deep on at least one component instead of staying at the boxes-and-arrows level for the whole hour.
- Always close the loop on: what happens when this component fails, and how do you observe/debug it in production.
