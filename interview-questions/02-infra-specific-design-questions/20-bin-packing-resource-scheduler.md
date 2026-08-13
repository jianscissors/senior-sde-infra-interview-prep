# Design a Bin-Packing / Resource Scheduler for a Compute Cluster

*(The "how does the Kubernetes scheduler actually decide" question, generalized.)*

## Clarifying questions to ask first
- What resource dimensions matter — CPU and memory only, or also GPU, disk, network bandwidth?
- Is the goal to optimize for cost (pack tightly, minimize idle capacity) or availability (spread workloads to reduce correlated-failure blast radius) — these pull in opposite directions?
- Do workloads have priority levels requiring preemption of lower-priority work?
- Static placement only, or does the scheduler need to handle live rebalancing as the cluster's load shifts over time?

## Requirements
### Functional
- Given a workload's resource request (CPU, memory, etc.) and a set of candidate machines with available capacity, place the workload on a machine that can satisfy it.
- Respect constraints: hard requirements (node has a GPU, workload needs a GPU) and soft preferences (spread across availability zones).

### Non-functional
- Scheduling decisions must be fast — this is on the critical path of workload startup latency across the whole cluster.
- Must handle a constantly changing cluster (nodes joining/leaving, capacity changing as other workloads start/stop) without requiring a full re-solve from scratch each time.

## High-level architecture
1. **Scheduler loop**: watches a queue of unscheduled workloads and the current state of cluster capacity; for each pending workload, runs a two-phase decision — **filtering** (which nodes are even eligible) then **scoring** (rank eligible nodes and pick the best).
2. **Cluster state store**: the source of truth for what capacity exists and what's already allocated where — must be kept accurate as workloads start, stop, and nodes' available capacity changes.
3. **Binding**: once a node is chosen, the assignment is committed (written to the state store) before the workload is actually started on that node, so a concurrent scheduling decision doesn't double-book the same capacity.

## Deep dives

### Bin-packing algorithm tradeoffs: best-fit vs spread
- **Best-fit / tight packing**: place each workload on the node where it leaves the least leftover capacity, maximizing overall cluster utilization and minimizing the number of nodes needed (directly reduces infrastructure cost). Downside: correlated failures — if that node dies, many tightly co-located workloads go down together, and tightly packed nodes leave little headroom to absorb a burst in any single workload's resource usage.
- **Spread**: distribute workloads as evenly as possible across nodes (and ideally across failure domains — racks, availability zones), maximizing availability and burst headroom at the cost of lower overall utilization (more idle capacity, more nodes needed, higher cost).
- Real schedulers (Kubernetes included) use a weighted scoring function that blends both goals rather than picking one exclusively — e.g., score partly on bin-packing efficiency and partly on spread across failure domains, with the weights tunable per cluster's priorities.

### Handling resource fragmentation over time
- As workloads of varying sizes start and stop, a cluster with plenty of *aggregate* free capacity can still fail to place a large workload because the free capacity is fragmented across many nodes in small pieces (classic bin-packing fragmentation). Mitigations: periodic **rebalancing** (proactively moving/rescheduling some workloads to consolidate free capacity), and bin-packing-biased placement in the first place (favor filling partially-full nodes over spreading onto empty ones, for workloads where packing is prioritized over spread).

### Priority and preemption
- Higher-priority pending workloads that can't otherwise be placed may **preempt** (evict) lower-priority running workloads to free up capacity — the scheduler needs a defined priority ordering and a policy for choosing which victim(s) to evict (typically: evict the minimum number of lowest-priority workloads needed to fit the new one, not an arbitrary set). Preempted workloads should be gracefully drained/rescheduled elsewhere, not just killed outright, where the workload type allows it.

### Multi-dimensional constraints
- Real scheduling isn't just "does this node have enough CPU" — it's a multi-dimensional bin-packing problem (CPU, memory, GPU, disk, network, and sometimes soft constraints like "must be in the same rack as service X" or "must not be on the same node as another replica of itself" for availability). The filtering phase eliminates any node failing a hard constraint on any dimension; the scoring phase then optimizes across the remaining dimensions and soft preferences simultaneously, typically via a weighted sum of per-dimension scores.

### Fairness across tenants sharing the cluster
- Without fairness controls, a single tenant submitting a large batch of workloads can monopolize free capacity and starve other tenants' pending work. Common approaches: per-tenant quotas (hard caps on resource share), or a fair-share scheduling algorithm that weights queue priority by how much capacity a tenant has already consumed relative to their entitlement.

## Key tradeoffs
- Tight bin-packing minimizes cost but maximizes correlated-failure blast radius and reduces burst headroom; spreading maximizes availability but costs more in idle capacity.
- A fast, greedy scheduling decision (score and place immediately) doesn't guarantee a globally optimal packing across the whole cluster; a batched/optimized approach (consider many pending workloads together) can pack tighter but adds latency to every individual placement decision.

## Failure modes
- Two concurrent scheduling decisions both target the same node's "last available slot" based on stale state: solved by committing the binding atomically against the state store (optimistic concurrency check, or a single-writer scheduling decision path) so a race doesn't double-book capacity.
- A node fails after workloads are already scheduled on it: those workloads need to be detected as no-longer-running (via a heartbeat/lease mechanism) and rescheduled elsewhere — this is a reconciliation loop, not a one-time placement decision.

## Likely follow-ups
- "How would you schedule for GPU workloads specifically, where GPUs can't be fractionally shared like CPU?" → treat GPU as a discrete, non-shareable resource dimension in the filtering phase (a node either has a free GPU slot or it doesn't — no fractional allocation the way CPU shares work), often requiring GPU-aware node labeling/taints so only GPU-requesting workloads are even considered for GPU nodes.
- "The scheduler itself needs to scale — what happens at very high scheduling throughput?" → parallelize scheduling across multiple scheduler instances partitioned by some dimension (e.g., by node pool or by workload priority class), while still serializing the final binding step per-node to avoid the double-booking race described above.
