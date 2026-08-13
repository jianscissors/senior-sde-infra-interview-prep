# Design a Service Discovery System (like Consul/etcd-backed discovery, or client-side discovery for gRPC)

## Clarifying questions to ask first
- Client-side discovery (clients query the registry and load-balance themselves) or server-side (a load balancer/proxy queries the registry on the client's behalf)?
- How fast must a newly started or crashed instance be reflected — seconds, or is a longer propagation window acceptable?
- DNS-based interface required (for compatibility with existing tooling) or is a custom API acceptable?

## Requirements
### Functional
- Services register themselves (or are registered) with network location (host:port) and metadata (version, zone, health).
- Clients (or a proxy on their behalf) can look up healthy instances of a given service by name.
- Unhealthy/dead instances are removed from results within a bounded time.

### Non-functional
- The registry itself must be highly available — if it goes down, it must not take down the ability of services to find each other for calls already in flight or recently cached.
- Low propagation latency for both new-instance-up and instance-down events.
- Scale to tens of thousands of service instances registering/deregistering/heartbeating continuously.

## High-level architecture
- **Registry store**: a strongly consistent, replicated key-value store (e.g., etcd/ZooKeeper using Raft) holding the live set of service → instance mappings.
- **Registration path**: each service instance registers itself on startup and sends periodic heartbeats; the registry expires an entry if heartbeats stop (lease/TTL model).
- **Health checking**: either the registry actively probes instances (HTTP/TCP health check), or instances self-report health, or both — active checking catches instances that go silent without deregistering cleanly (crash, network partition).
- **Discovery path**: clients query the registry directly (client-side discovery, common with gRPC + a resolver plugin), or a proxy/load balancer (e.g., Envoy) queries it and clients just talk to the proxy (server-side discovery, simpler clients, extra network hop).

## Deep dives

### Registration: self-register with heartbeat vs sidecar-managed
- **Self-registration**: the service process registers itself directly with the registry on startup and sends its own heartbeats. Simple, no extra infrastructure, but couples every service to the registry's client library and puts registration correctness in each team's hands.
- **Sidecar-managed** (e.g., in a service mesh): a co-located sidecar proxy handles registration/heartbeating on the service's behalf, based on orchestrator signals (e.g., the pod being marked Ready by Kubernetes). Decouples app code from discovery entirely and centralizes the failure-handling logic — the tradeoff is operational dependency on the sidecar itself being correctly deployed everywhere.

### Health-checking integration
- Heartbeat/TTL alone isn't sufficient — a process can be alive (still sending heartbeats) but functionally unhealthy (e.g., stuck, can't serve traffic). Layer in active health checks (HTTP `/healthz` polling) so the registry reflects actual serviceability, not just process liveness.
- Decide what "unhealthy" removes: immediate removal from discovery results on a single failed check risks flapping instances in/out under transient blips; most systems require N consecutive failures before removal, and symmetrically M consecutive successes before re-adding.

### Propagation latency vs consistency tradeoff
- Fully consistent, synchronous propagation (every client always sees the exact current state) is expensive to guarantee at scale and adds latency. Most service discovery systems instead accept **eventual consistency with a bounded staleness window**: clients cache the last-known-good instance list and poll/watch for updates, tolerating a few seconds of staleness in exchange for not hammering the registry on every single call.
- This means a freshly dead instance may still receive a small amount of traffic for a short window after failure — client-side retry logic (fail over to a different instance on error) is the real safety net here, not perfect discovery-layer accuracy.

### Client-side caching with fallback on registry unavailability
- Clients should cache the last resolved instance list locally and continue using it (with periodic background refresh attempts) if the registry becomes temporarily unreachable, rather than failing every outbound call. This is what keeps a registry outage from cascading into a full platform outage — the registry is a control-plane dependency, and control-plane unavailability should degrade gracefully, not take down the data plane.

### DNS-based vs API-based discovery
- **DNS-based**: works with zero client-side code changes (standard DNS resolution), broadly compatible, but constrained by DNS's caching/TTL semantics and typically can't carry rich metadata (zone, version, weights) beyond an IP list.
- **API-based**: a purpose-built API (gRPC/HTTP) returns rich metadata and supports push-based updates (long-poll or streaming watch), enabling much faster propagation than DNS TTL-bound caching allows — the standard choice inside a service mesh or when using gRPC's pluggable resolver interface.

## Key tradeoffs
- Client-side discovery avoids an extra network hop and a proxy fleet to operate, but pushes discovery logic (and its failure handling) into every client/language. Server-side discovery centralizes complexity into one well-tested proxy layer at the cost of an extra hop's latency.
- Strong consistency in the registry is achievable (Raft-backed store) but propagation to thousands of watching clients is where eventual consistency creeps back in regardless — be explicit about which layer is strongly consistent and which isn't.

## Failure modes
- Registry cluster loses quorum: existing registrations already cached by clients keep working; new registrations/deregistrations block until quorum is restored — this is why client-side caching with fallback matters so much.
- An instance crashes without deregistering: heartbeat TTL expiry (typically tens of seconds) is the backstop that eventually removes it; active health checks catch "alive but broken" faster than TTL expiry alone would.

## Likely follow-ups
- "How do you avoid thundering-herd load on the registry when thousands of clients all refresh at once?" → jittered polling intervals, or push-based watch streams instead of polling, so updates fan out incrementally rather than in synchronized bursts.
- "How would you handle discovery across multiple regions/datacenters?" → per-region registry replicas with cross-region async replication for global visibility, but keep same-region lookups fast and locally served so cross-region latency/partitions don't block local traffic.
