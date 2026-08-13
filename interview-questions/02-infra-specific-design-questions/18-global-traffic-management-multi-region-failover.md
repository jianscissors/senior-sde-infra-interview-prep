# Design a Global Traffic Management / Multi-Region Failover System

## Clarifying questions to ask first
- Active-active (all regions serve live traffic) or active-passive (one primary, others standby)?
- What's the data layer's replication topology — does data replicate across regions synchronously or asynchronously?
- What RTO/RPO (recovery time / recovery point objective) is acceptable for a regional failure?
- Should failover be fully automated, or does a human need to approve it given the risk?

## Requirements
### Functional
- Route users to a healthy region under normal conditions (usually the geographically closest, for latency).
- Detect a region becoming unhealthy and redirect traffic away from it within an acceptable time window.

### Non-functional
- Health detection must reliably distinguish a real regional outage from a transient blip, to avoid flapping traffic between regions.
- Failover must not violate data-consistency guarantees the application depends on.

## High-level architecture
1. **Health-checking layer**: active probes against each region's endpoints (from multiple external vantage points, not just from one location, to avoid a single bad network path looking like a regional outage).
2. **Traffic routing layer**: DNS-based global server load balancing (GSLB) and/or an anycast/edge routing layer that directs client requests to a region based on health + proximity + capacity.
3. **Data layer**: cross-region replication strategy that determines what "failing over" actually means for correctness (see below).

## Deep dives

### Health-checking each region
- Probe multiple independent signals per region (load balancer health, a synthetic end-to-end transaction, key dependency health) rather than a single shallow ping — a region can respond to ICMP/TCP checks while its application layer is actually failing. Probe from several external locations to distinguish "this region is down" from "there's a network problem between one prober and this region."

### DNS/GSLB-based routing and TTL tradeoffs
- DNS-based failover changes which IP a domain resolves to, but every resolver and client caches that answer for the record's TTL. A long TTL (hours) means stale, cached, wrong answers persist long after you've flipped the routing — but a very short TTL (seconds) multiplies DNS query volume and load on the DNS infrastructure itself. Practical failover TTLs are usually in the tens-of-seconds to low-minutes range, balancing failover speed against query load — and even then, some fraction of clients/resolvers will ignore TTL and cache longer, so DNS failover alone is never instantaneous for 100% of traffic.
- An anycast-routed edge layer (route via BGP to the nearest healthy point of presence) avoids the DNS-caching problem entirely and can fail over sub-second, at the cost of needing that anycast/edge infrastructure in the first place.

### Active-active vs active-passive
- **Active-active**: all regions serve live traffic simultaneously; failover just means routing away from the unhealthy region toward regions already warm and already handling load — fastest recovery, but requires the data layer to support multi-region writes (harder consistency problem) and requires every region to be provisioned for more than its "fair share" of load so it can absorb another region's traffic on failover.
- **Active-passive**: one primary region takes all writes; standby region(s) replicate but don't serve live write traffic. Simpler data consistency story, but failover means promoting a standby, which takes longer and that standby may need to scale up capacity it wasn't previously serving.

### Data replication lag's effect on failover safety
- If replication to the standby/other-region is asynchronous, failing over to it after the primary dies means losing any writes that hadn't replicated yet — this is exactly what RPO (recovery point objective) quantifies. Failing over blindly the instant a region looks unhealthy risks failing over to a region with meaningfully stale data, which can be worse for the business than a slower, deliberate failover with a data-integrity check first.

### Automated vs human-gated failover
- Automating failover minimizes downtime but risks acting on a false-positive health signal (flapping traffic between regions, or failing over unnecessarily and eating the replication-lag data-loss risk for no reason) and risks a bad automated decision compounding a partial outage into a full one. A common middle ground: automate failover for clearly severe, unambiguous outages (all health signals red across all vantage points) but require human confirmation for ambiguous or borderline cases — and always automate the fast, low-risk direction (routing traffic back to a now-healthy region, which is like restoring cache — safe to retry).

## Key tradeoffs
- Active-active gives faster failover and better resource utilization but forces a harder, and usually eventually-consistent, data model across regions.
- Fast automated failover reduces downtime but increases the risk of an unnecessary or badly-timed failover if the health signal is noisy; slower human-gated failover is safer per-incident but adds guaranteed downtime for every real incident while waiting for a human.

## Failure modes
- **Split-brain**: if a network partition makes two regions each believe they're the sole primary and both accept writes, data diverges — this is why active-active designs need either a conflict-resolution strategy (e.g., last-write-wins, CRDTs) or a single-writer arbitration mechanism (e.g., a global lock/leader election across regions) rather than assuming clean failover.
- Failing over to a region that isn't actually capacity-provisioned for full traffic: the "successful" failover just moves the outage (now it's a capacity outage in the new region) — capacity planning for the failover target is part of the design, not an afterthought.

## Likely follow-ups
- "How do you test that failover actually works, before you need it in a real incident?" → regular game-day exercises that force a real failover in production (or a close facsimile) on a schedule, not just a design doc — untested failover paths are notoriously likely to fail exactly when needed.
- "What's the difference between failing over DNS vs failing over at the load balancer / anycast layer?" → DNS failover is simple and works everywhere but is bounded by client/resolver caching behavior; anycast/edge-layer failover is faster and more precise but requires owning that network infrastructure.
