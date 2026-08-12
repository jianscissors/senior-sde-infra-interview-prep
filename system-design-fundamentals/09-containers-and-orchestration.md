# Containers & Orchestration

## Why it comes up
This is the most infra-specific fundamentals topic — expect it to appear both as direct questions ("how does Kubernetes scheduling work?") and as an assumed substrate for other designs ("your service runs on K8s, how do you handle a rolling deploy?").

## Containers vs VMs
- **VM:** full guest OS + hypervisor virtualizes hardware. Strong isolation, heavier (seconds-to-minutes boot, GBs of overhead).
- **Container:** shares the host kernel, isolated via Linux namespaces (PID, network, mount, UTS, IPC, user) and resource-limited via cgroups. Lightweight (ms-to-seconds start, MBs of overhead), weaker isolation boundary than a VM (kernel is shared — a container escape can affect the host).
- **Image layers:** union filesystem (overlayfs) — each layer is a diff, layers are cached/shared across images, enables fast pulls when base layers already exist locally.

## Kubernetes core objects (know the vocabulary fluently)
- **Pod:** smallest deployable unit, one or more containers sharing network namespace/IP and storage volumes.
- **Deployment:** manages a ReplicaSet of pods, handles rolling updates/rollbacks declaratively.
- **Service:** stable virtual IP + DNS name that load-balances to a dynamic set of pod IPs (via label selectors); `ClusterIP` (internal), `NodePort`, `LoadBalancer` (cloud LB provisioning), `Headless` (direct pod DNS, no VIP — used for StatefulSets).
- **StatefulSet:** for stateful workloads needing stable network identity and stable storage per replica (ordered, sticky pod names like `pod-0`, `pod-1`).
- **DaemonSet:** one pod per node (log collectors, node monitoring agents).
- **ConfigMap/Secret:** externalized config/credentials injected as env vars or mounted files.
- **Ingress / Gateway API:** L7 routing rules (host/path-based) from outside the cluster to Services.
- **Namespace:** logical isolation/multi-tenancy boundary within a cluster (RBAC and resource quotas apply per namespace).

## Scheduling
- Scheduler binds unscheduled pods to nodes based on: resource requests/limits (CPU/memory fit), affinity/anti-affinity rules (co-locate or spread pods), taints/tolerations (repel pods from nodes unless explicitly tolerated — e.g., dedicated/tainted nodes for GPU workloads), topology spread constraints (spread across zones/racks for availability).
- **Requests vs limits:** requests are what the scheduler reserves (guaranteed), limits are the hard ceiling (throttled for CPU, OOM-killed for memory if exceeded) — overcommitting limits > requests is common and risky under real contention.
- Control loop model: Kubernetes is fundamentally **reconciliation loops** — controllers continuously compare desired state (from etcd, the source of truth) to observed actual state and act to converge them; this "level-triggered not edge-triggered" design is why K8s self-heals after arbitrary failures (it doesn't need to know what went wrong, just re-converges).

## Deployment strategies
- **Rolling update:** replace old pods with new gradually, controlled by `maxSurge`/`maxUnavailable` — default, minimal extra capacity needed, but old and new versions coexist during rollout (must be backward/forward compatible, especially for schema changes).
- **Blue-green:** two full environments, switch traffic atomically — instant rollback, but 2x resource cost during transition.
- **Canary:** shift a small % of traffic to the new version, monitor, then ramp — catches regressions with limited blast radius, needs good metrics/automation to be safe.
- **Feature flags:** decouple deploy from release — code ships dark, enabled progressively/independently of the deploy pipeline.

## Networking in Kubernetes
- Every pod gets its own IP (flat network model — pods can reach each other directly without NAT, a core K8s networking requirement), implemented by a CNI plugin (Calico, Cilium, etc.).
- kube-proxy (or eBPF-based equivalents like Cilium) implements Service VIPs by programming iptables/IPVS/eBPF rules that load-balance to backing pod IPs.
- **Service mesh** (Istio, Linkerd): sidecar proxies (Envoy) intercept all pod traffic to add mTLS, retries/timeouts/circuit-breaking, fine-grained traffic splitting, and observability uniformly without changing app code — the tradeoff is added latency per hop and operational complexity of the mesh control plane itself.

## Common follow-ups
- "How does Kubernetes decide a node/pod is unhealthy?" → liveness probes (restart the container), readiness probes (remove from Service endpoints, don't restart), startup probes (delay the others during slow boot).
- "What happens during a rolling deploy if the new version has a bug?" → readiness probe failures halt the rollout automatically, but "halted" isn't "rolled back" — that's automation you build (progressive delivery tools like Argo Rollouts/Flagger watch metrics and auto-rollback).
- "How would you safely run a stateful database on Kubernetes?" → StatefulSet + PersistentVolumeClaims (durable storage independent of pod lifecycle) + careful pod disruption budgets + often still prefer a managed/dedicated approach for the hardest state (many infra orgs deliberately keep core databases off k8s or on dedicated node pools).
