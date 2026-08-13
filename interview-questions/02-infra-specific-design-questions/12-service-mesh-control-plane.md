# Design a Service Mesh's Control Plane (like a simplified Istio control plane)

## Clarifying questions to ask first
- Scale: how many sidecar proxies (thousands? tens of thousands?) and how frequently does config change?
- Do we need mTLS/certificate management in scope, or just traffic routing config?
- Is the data plane (per-pod sidecar proxy) assumed to already exist, with this question focused purely on the control plane that configures it?

## Requirements
### Functional
- Distribute routing rules, retry/timeout/circuit-breaking policy, and traffic-split (canary) config to every sidecar proxy in the mesh.
- Issue and rotate mTLS certificates for service-to-service authentication.
- Reflect service discovery/endpoint changes (pods coming up/down) into proxy configuration.

### Non-functional
- Configuration propagation must scale to thousands of proxies without the control plane becoming a bottleneck or single point of failure.
- The data plane (actual traffic flow) must keep functioning even if the control plane is temporarily degraded or unreachable — this is the most important non-functional property of the whole design.
- Config updates should propagate with low latency (seconds, not minutes) — a stale routing rule or an unrevoked certificate is a real security/correctness risk.

## High-level architecture
1. **Config store**: source of truth for mesh-wide configuration (routing rules, policies), typically backed by the underlying orchestrator's API (e.g., Kubernetes CRDs) or a dedicated config service.
2. **Control plane (e.g., Istiod)**: watches the config store and cluster state (service discovery/endpoints), translates high-level policy into proxy-specific configuration, and pushes it to each sidecar.
3. **Data plane (sidecar proxies, e.g., Envoy)**: receives config via a discovery protocol, applies it locally, and handles all actual traffic — the control plane is never in the data path of a single request.
4. **Certificate authority**: issues short-lived workload certificates to each sidecar for mTLS, handling rotation before expiry.

## Deep dives

### Config propagation to thousands of sidecars without overwhelming the control plane
- Naive full-config-push to every proxy on every change doesn't scale — both because the payload is large and because a burst of simultaneous changes (e.g., a large deployment rolling out) could create a thundering herd of full pushes. The standard solution (used by Envoy's xDS protocol) is **incremental/delta updates**: after an initial full sync, only the specific resources that changed are pushed, keyed by resource version, dramatically cutting the size and frequency of updates under normal churn.
- The control plane also shards/batches pushes and uses backpressure-aware streaming (a long-lived gRPC stream per proxy) rather than a request-response model per update, so it can push updates to many proxies concurrently without needing a full round of synchronous acks before proceeding.

### xDS-style incremental updates
- xDS (the family of discovery APIs: LDS for listeners, RDS for routes, CDS for clusters, EDS for endpoints) lets the control plane push exactly the resource type that changed (e.g., just an endpoint set update when a pod restarts) rather than the full mesh config, and lets proxies ACK/NACK individual updates so the control plane knows precisely which proxies are on which config version — critical for debugging "why is this one proxy behaving differently" in a large mesh.

### Certificate rotation at scale
- Each workload gets a short-lived certificate (e.g., valid for hours, not the year+ validity typical of traditional TLS certs) issued by the mesh's CA, tied to its service identity (e.g., a SPIFFE ID). Short lifetimes limit the blast radius of a leaked cert automatically (it expires soon regardless of revocation), but require the control plane to reliably rotate certs before expiry for every workload continuously — a rotation failure that goes unnoticed becomes an outage once certs actually expire, so rotation health itself needs monitoring/alerting.

### Blast radius if the control plane is degraded
- This is the property that most distinguishes a well-designed mesh: sidecar proxies cache their last-received configuration locally and keep operating on it if the control plane becomes unreachable — "control plane down" must not mean "data plane down," since the data plane already has everything it needs to keep routing traffic. The risk window is that config becomes stale (a new canary rule or a cert rotation won't apply) until the control plane recovers, which is an acceptable degradation compared to a full mesh-wide traffic outage.
- Certificates are the sharp edge here: if the control plane is down long enough for certs to actually expire before rotation completes, mTLS connections will start failing — this is why cert lifetimes need to be long enough to survive a reasonable control-plane outage window, not so short that a short outage causes a cascading auth failure.

## Key tradeoffs
- Short-lived certs improve security (fast automatic revocation via expiry) but raise the operational stakes of control-plane availability, since rotation has a hard deadline.
- Push-based incremental config (xDS) is far more efficient at scale than proxies polling for full config on an interval, but is more complex to implement correctly (tracking per-proxy ACK state, versioning, ordering).

## Failure modes
- Control plane process crashes/restarts: sidecars keep serving traffic on cached config; new pods starting during the outage may get default/fallback config until the control plane recovers and can push real config to them.
- A bad config push (e.g., a broken routing rule) rolls out mesh-wide: mitigated by canarying config changes to a subset of proxies first, the same discipline applied to code deploys — a config push is a deploy and should be gated the same way.

## Likely follow-ups
- "How do you debug 'proxy A and proxy B have different behavior' in a 5,000-proxy mesh?" → xDS ACK/NACK + per-proxy config version reporting lets you query exactly what config version each proxy currently has applied, rather than guessing.
- "What happens to in-flight requests during a certificate rotation?" → rotation swaps the cert used for new connections; existing established mTLS connections aren't torn down mid-flight, only new connection setups use the rotated cert, avoiding disruption to active traffic.
