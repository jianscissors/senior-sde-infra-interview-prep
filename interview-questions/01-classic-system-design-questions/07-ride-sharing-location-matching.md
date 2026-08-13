# Design a Ride-Sharing / Location-Based Matching System (e.g., Uber Dispatch)

## Clarifying questions to ask first
- How frequently do driver locations update (every few seconds via GPS ping)?
- What's the matching objective — nearest driver, or a more sophisticated ETA/fairness-aware match?
- Do we need to handle surge pricing / demand-supply imbalance as part of the design?
- Geographic scope — a single dense city, or global with wildly varying driver density?

## Requirements
**Functional**
- Riders request a ride and get matched to a nearby available driver.
- Drivers continuously report location; the system tracks who's available.
- Both parties see real-time location updates during the trip.

**Non-functional**
- Matching latency should be low (a few seconds) — riders won't tolerate long waits for a match.
- Must handle extreme geographic hot spots (e.g., downtown at rush hour, a stadium after an event) without degrading.
- Location updates arrive at high, continuous volume from every active driver.

## Back-of-envelope estimation
- 5M active drivers, each pinging location every 4 seconds → ~1.25M location writes/sec globally — this write volume, not the matching itself, is often the biggest scaling challenge.
- A dense city center might have thousands of drivers within a few square kilometers — geospatial queries must handle this density without scanning all of them per request.

## High-level architecture
1. **Location ingestion**: drivers stream GPS pings to a location service, which updates a geospatial index of current driver positions.
2. **Matching service**: on a ride request, query the geospatial index for nearby available drivers, rank candidates, and dispatch a match request.
3. **Trip service**: tracks trip state (requested → matched → in progress → completed) and streams live location to both rider and driver during the trip.

## Deep dives

### Geospatial indexing (geohash, quad-tree, S2)
- **Geohash**: encodes lat/lng into a string where common prefixes indicate spatial proximity; querying "drivers near me" becomes a prefix range query, which maps naturally onto a sorted key-value store or a database index. Simple, but has edge cases at cell boundaries (two nearby points can have very different geohash prefixes if they straddle a boundary) and non-uniform cell sizes near the poles.
- **Quad-tree**: recursively subdivides space into quadrants, with denser regions subdivided further — naturally adapts to non-uniform driver density (fine-grained cells downtown, coarse cells in sparse suburbs). More complex to shard across machines than a geohash's flat key space.
- **S2 geometry** (Google's library): projects the sphere onto a cube and indexes cells hierarchically, avoiding the lat/lng distortion issues of geohash near the poles and giving more uniform cell-area properties. This is the answer that signals real depth if the interviewer pushes past geohash.
- In all cases, the core query is "give me candidates within radius R," which the geospatial structure turns into a small number of index lookups instead of a full scan.

### Real-time location updates at scale
- Driver location writes are extremely high volume but each individual update largely supersedes the last (you mostly care about *current* location, not full history for matching purposes) — this favors an in-memory store (Redis with geospatial commands, or a custom in-memory geo-index) over a durable database write per ping, with periodic/async persistence for historical trip records only.
- Shard the location index geographically (e.g., by geohash prefix or S2 cell region) so writes and nearby-driver queries for a given area stay local to one shard, rather than a global index single point of contention.

### Matching algorithm
- Naive nearest-driver matching can be locally greedy but globally suboptimal (e.g., assigning the closest driver to rider A when that driver would better serve rider B who's also waiting nearby, leaving a worse match for both). More sophisticated systems batch nearby unmatched requests over a short window (a few seconds) and solve a small-scale assignment optimization (minimizing total ETA) rather than matching strictly first-come-first-served — a reasonable tradeoff to mention even if you implement the simpler greedy version first.
- Once a candidate driver is selected, the match is a *request* (driver can accept/decline/timeout), not an assignment — the matching service needs a short-lived reservation/lock on that driver so two simultaneous ride requests don't both dispatch to the same driver before either accepts.

### Handling hot geographic areas
- A stadium letting out or a dense downtown core can create a massive spike in both ride requests and driver density in one small area — the geospatial shard covering that area can become a hot shard. Mitigate with finer-grained sharding in known dense areas (denser quad-tree subdivision, or dynamic re-sharding based on observed load) rather than a fixed uniform grid.
- Demand-supply imbalance in a hot area is also where surge pricing comes in as a supply-side signal (incentivize more drivers into the area) — worth a one-line mention even if it's out of scope for the core design.

## Key tradeoffs
- In-memory, eventually-persisted location data (fast, some data loss risk on crash) vs. durable writes per ping (safe, but far too slow/expensive at this write volume) — real systems accept the eventual-persistence tradeoff for live location, since a missed historical ping is low-stakes compared to matching latency.
- Strict first-come-first-served matching (simple, fast) vs. batched optimization matching (better global outcomes, adds latency and complexity) — most interviews are satisfied if you propose greedy-first and can articulate the batching upgrade as a follow-up.

## Failure modes
- Location service for a geographic shard becomes unavailable → matching in that region should degrade to a coarser fallback (e.g., query an adjacent shard or a slightly stale replica) rather than failing ride requests outright.
- A matched driver never responds (app crashed, no signal) → the reservation must have a timeout, after which the ride re-enters the matching pool and is offered to the next candidate — don't let a single unresponsive driver block a rider indefinitely.

## Likely follow-ups
- "How would you show the rider their driver moving smoothly on the map, not jumping between GPS pings?" → client-side interpolation between received points, not a server-side concern.
- "How do you handle a driver going offline mid-trip (tunnel, dead zone)?" → trip state shouldn't depend on continuous connectivity; treat trip progress as resumable from the last known state, and use timeouts/heartbeats to distinguish "temporarily out of range" from "actually disconnected."
