# Design an Autoscaler for a Fleet of Services

## Clarifying questions to ask first
- Reactive scaling (respond to current load) or do we also need predictive/scheduled scaling for known traffic patterns (e.g., daily peaks)?
- What metrics drive the decision — CPU, memory, request queue depth, custom business metrics (e.g., pending jobs)?
- Scale-out and scale-in are usually asymmetric in risk — should they use different policies?
- Is this scaling stateless app servers, or does it also need to handle stateful workloads (harder — can't just kill an instance mid-work)?

## Requirements
### Functional
- Continuously observe a signal (metric) per service and add/remove instances to keep the signal within a target range.
- Respect configured min/max instance bounds per service.

### Non-functional
- Scale-out must be fast enough to absorb a traffic spike before it causes user-facing degradation.
- Must avoid **flapping** — rapidly oscillating scale-out/scale-in decisions that thrash the fleet and its dependencies (load balancer registration churn, cold-start costs, connection draining overhead).

## High-level architecture
1. **Metrics source**: the monitoring pipeline (see the metrics/monitoring design) feeds current utilization per service.
2. **Controller loop**: runs on an interval (e.g., every 15-30s), compares current metric value against the target, computes a desired instance count, and issues scale-out/scale-in commands to the underlying compute layer (VM group, container orchestrator).
3. **Execution layer**: actually provisions/terminates instances, registers/deregisters them with the load balancer, and drains connections before terminating on scale-in.

## Deep dives

### Metrics-driven scaling decisions
- CPU/memory are simple but often lag the real bottleneck — a service can be I/O-bound or blocked on a downstream dependency while CPU stays low. **Queue depth** (pending requests/jobs) or a custom application-level metric (e.g., in-flight request count) is often a better leading indicator, since it reflects actual work backing up rather than a proxy resource metric.
- Desired instance count is typically computed proportionally: `desired = current_instances * (current_metric / target_metric)`, then clamped to configured min/max.

### Avoiding flapping
- **Cooldown periods**: after a scaling action, wait a fixed interval before allowing another action in the same direction, giving the fleet time to actually absorb the change and the metric time to stabilize before reacting again.
- **Hysteresis**: use different thresholds for scale-out vs scale-in (e.g., scale out above 70% CPU, scale in only below 30% CPU) rather than a single target, so noise around one threshold doesn't cause continuous back-and-forth.
- Aggregate the metric over a short window (e.g., average over the last 3-5 data points) rather than reacting to a single instantaneous reading, to smooth out transient spikes.

### Scale-out speed vs scale-in caution
- Scale-out should be fast and biased toward over-provisioning slightly — the cost of a spare instance is much lower than the cost of a latency spike or dropped requests during a real traffic surge.
- Scale-in should be conservative and gradual — remove a small number of instances at a time, re-evaluate, and always drain connections (stop routing new traffic, let in-flight requests finish, then terminate) rather than hard-killing instances, which would drop in-flight work.

### Predictive/scheduled scaling
- For known, recurring traffic patterns (daily peak at 9am, weekly batch job spikes), pre-scale ahead of the predicted spike on a schedule rather than waiting for reactive metrics to catch up — reactive scaling alone always has a lag between load increasing and new capacity coming online (instance boot time, warm-up, health-check pass).

### Interaction with load balancer health checks and connection draining
- A newly scaled-out instance must pass health checks and be registered with the load balancer before it receives traffic — don't count it as "capacity" until it's actually routable, or the autoscaler will under-count effective capacity and over-scale.
- On scale-in, deregister the instance from the load balancer *first*, wait for existing connections to drain (bounded by a max drain timeout), and only then terminate — terminating immediately drops in-flight requests.

## Key tradeoffs
- Aggressive scale-out thresholds cost more (idle capacity, and over-scaling risk) but reduce the chance of user-facing latency during spikes; conservative thresholds save cost but risk degraded performance until the autoscaler catches up.
- Reactive-only scaling is simpler to build and reason about; predictive scaling requires historical traffic modeling but eliminates the "always behind the spike" lag for known patterns.

## Failure modes
- Metrics pipeline is down or delayed: the autoscaler has no signal — default to holding current capacity (don't scale to zero, don't scale wildly) rather than acting on stale or missing data.
- A downstream dependency (e.g., database) can't handle the fleet scaling out as far as the metric would suggest: autoscaling one service can overload a shared downstream dependency — cap max instances with the downstream capacity in mind, or coordinate scaling limits across the call graph.

## Likely follow-ups
- "What happens if the metric itself is caused by an autoscaling bug (e.g., a broken health check keeps failing instances, triggering endless scale-out)?" → enforce a hard max-instance ceiling and alert loudly when it's hit, rather than scaling unboundedly; treat "hit max scale" as an incident signal.
- "How do you scale a stateful service (e.g., a database shard) safely?" → you generally can't autoscale stateful instances the same way — scale-in requires safely migrating or draining state first, so stateful workloads usually need a different, slower, more orchestrated scaling process (or you scale a stateless layer in front of a fixed stateful backend instead).
