# Scalability & Load Balancing

## Why it comes up
Nearly every design question starts here: how do you take a single-server design and make it handle 10x/100x/1000x the load. Infra interviews expect you to reason about this quantitatively, not just say "add more servers."

## Core concepts

### Vertical vs horizontal scaling
- **Vertical (scale up):** bigger machine. Simple, but has a ceiling, single point of failure, downtime to resize, gets expensive non-linearly.
- **Horizontal (scale out):** more machines. Requires statelessness (or partitioning of state), a way to distribute load, and coordination. This is the default answer for infra roles.

### Back-of-envelope estimation (do this out loud in interviews)
- QPS = daily active users × actions/user / 86,400s. Multiply by 2-3x for peak/day skew.
- Storage = records/day × size/record × retention. Convert to TB/PB and sanity check.
- Bandwidth = QPS × payload size.
- Rule of thumb: a well-tuned server can handle ~1k-10k simple requests/sec; DB writes are much lower (~1k-5k/sec per node depending on engine/hardware).

### Load balancing
- **L4 (transport layer):** routes on IP/port, doesn't inspect HTTP. Fast, protocol-agnostic. Example: AWS NLB, IPVS/LVS.
- **L7 (application layer):** routes on HTTP headers/path/cookies. Enables content-based routing, TLS termination, retries. Example: ALB, Envoy, Nginx, HAProxy.
- **DNS-based (GSLB):** routes at the DNS level across regions/data centers. Coarse-grained, TTL-limited, used for geo-routing and failover.
- **Client-side LB:** the client (or a sidecar) picks the backend itself using a service registry (e.g., gRPC client-side LB, Netflix Ribbon-style, Envoy sidecar). Removes a hop and a bottleneck at very high scale.

### LB algorithms
- Round robin / weighted round robin
- Least connections / least response time
- Consistent hashing (sticky routing without a central session store — see caching doc)
- Random with power-of-two-choices (very effective, low overhead, used by many modern proxies)

### Health checks & failure handling
- Active health checks (periodic probe) vs passive (mark down after N failed real requests).
- Outlier detection / circuit breaking to eject unhealthy backends automatically.
- Connection draining on deploy/scale-down so in-flight requests finish.

### Statelessness
Horizontal scaling requires pushing state out of the app server: sessions in a shared cache/store (or JWT), sticky routing only as an optimization not a requirement, idempotent request handling so retries/duplicates are safe.

### Autoscaling
- Reactive (CPU/queue-depth/latency-based) vs predictive/scheduled.
- Scale-out is usually fast; scale-in must be careful (drain connections, respect min-in-service, avoid flapping — use cooldowns/hysteresis).
- Autoscaling infra itself (e.g., the control plane, the LBs) often needs pre-provisioned headroom because it can't bootstrap itself under load.

## Common follow-ups
- "What happens when one region goes down?" → failover, health-check-driven DNS/GSLB, active-active vs active-passive.
- "How do you avoid thundering herd on scale-up?" → gradual traffic shifting, connection warm-up, jittered retries.
- "How do you load balance stateful connections (WebSockets, gRPC streams)?" → consistent hashing or client-side LB with long-lived connection awareness; L7 proxies need special handling since a single connection maps to one backend for its lifetime.
