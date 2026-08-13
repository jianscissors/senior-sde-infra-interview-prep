# Design a Distributed Unique ID Generator (e.g., Twitter Snowflake)

## Clarifying questions to ask first
- Must IDs be strictly monotonically increasing globally, or just roughly time-ordered and unique?
- What's the required throughput (IDs/sec) per node and system-wide?
- Can we tolerate a small amount of centralized coordination, or must generation be fully coordination-free per node?
- Do IDs need to be usable as database primary keys / sort keys (affects whether rough time-ordering is a requirement, not just a nice-to-have)?

## Requirements
**Functional**
- Generate a unique ID for every request, with no collisions across all nodes in the system.
- IDs should be roughly sortable by creation time (useful as primary/sort keys, and for debugging/ordering).

**Non-functional**
- Very high throughput, low latency generation (this is often on the critical path of every write).
- No single point of failure or coordination bottleneck at generation time.
- Must work correctly across many nodes generating IDs concurrently and independently.

## Back-of-envelope estimation
- A system doing 100K writes/sec system-wide needs ID generation to comfortably clear that rate with margin — any design requiring a network round-trip per ID (e.g., a centralized counter service) adds latency and a scaling bottleneck that a purely local generation scheme avoids entirely.

## High-level architecture
Two fundamentally different approaches, worth presenting as the core of the answer:
1. **Coordination-free (Snowflake-style)**: each ID is composed locally from a timestamp, a machine/worker ID, and a per-millisecond sequence number, bit-packed into a single integer. No network call needed to generate an ID.
2. **Coordinated (centralized counter service)**: a dedicated service (or database sequence) hands out ranges of IDs or increments a shared counter atomically. Simpler ID format (just an increasing integer), but adds a network hop and a shared dependency.

## Deep dives

### Snowflake-style bit layout
- Classic layout (64-bit ID): 1 unused sign bit + 41 bits timestamp (milliseconds since a custom epoch, giving ~69 years of range) + 10 bits machine/worker ID (supports 1024 distinct generators) + 12 bits per-millisecond sequence number (4096 IDs per machine per millisecond before needing to wait for the next millisecond).
- This packs *time*, *origin*, and *disambiguation* into one integer, generated entirely from local state — no coordination with any other node required for the common case.

### Monotonicity requirements
- IDs are monotonically increasing *per machine* (timestamp + incrementing sequence), but not strictly globally monotonic across machines unless clocks are perfectly synchronized — two different machines can generate IDs in the same millisecond where the "later" real-world event gets a numerically smaller ID if its machine's clock is slightly behind. For most use cases (rough time-ordering, uniqueness, usable as a sort key) this is acceptable; if strict global monotonicity is a hard requirement, that pushes you toward the coordinated approach instead.

### Clock drift handling
- The timestamp component depends on each machine's local clock being reasonably accurate (NTP-synced) — if a machine's clock jumps backward (NTP correction, VM migration), it risks generating an ID smaller than one it already generated, breaking the "roughly increasing" property and potentially causing a same-sequence collision within the same millisecond bucket.
- Standard mitigation: on startup (or on detecting the local clock has moved backward relative to the last-recorded generation timestamp), the generator refuses to issue IDs until the clock catches back up (or explicitly errors out), rather than silently risking a collision.

### Coordination-free vs. coordinated generation
- **Coordination-free** (embed timestamp + machine ID + sequence): scales linearly with the number of machines, no shared bottleneck, but requires assigning each generator a unique machine ID (itself a small coordination problem, usually solved once at startup via a config value, a ZooKeeper/etcd-assigned slot, or a Kubernetes pod ordinal — not per-ID).
- **Coordinated** (centralized counter, e.g., a database sequence or a Redis `INCR`): produces simpler, strictly monotonic IDs, but the counter service becomes a shared dependency in the request path and a potential bottleneck/single point of failure — commonly mitigated by handing out ID *ranges* to each client in batches (e.g., "you own IDs 1000-1999, generate locally within that range, come back for the next batch when exhausted") rather than round-tripping per ID.

## Key tradeoffs
- Coordination-free generation trades strict global monotonicity for horizontal scalability and no shared dependency — this is almost always the right trade for a high-throughput distributed system, which is why Snowflake-style IDs are the standard answer.
- 64-bit vs. 128-bit (e.g., UUID) IDs: Snowflake-style 64-bit IDs are compact and index-friendly (good for database primary keys, since sequential-ish values keep B-tree inserts localized rather than scattered); random UUIDs avoid needing a machine-ID assignment scheme at all but are worse for index locality and carry no time-ordering information.

## Failure modes
- A machine's clock skews backward → without the safeguard above, it risks issuing a duplicate/out-of-order ID; the generator should detect this and refuse to generate (fail loudly) rather than silently risk a collision.
- Machine-ID assignment collides (two nodes accidentally configured with the same machine ID) → this breaks the uniqueness guarantee entirely; machine ID assignment must itself be made reliably unique (e.g., derived from a coordinated source like a ZooKeeper/etcd sequence node at startup, or a cloud-provider-guaranteed unique instance identifier).

## Likely follow-ups
- "What happens when the per-millisecond sequence number overflows (more than 4096 IDs needed in one millisecond on one machine)?" → the generator busy-waits (or sleeps) until the next millisecond tick before continuing, briefly throttling that machine's generation rate — worth naming as the explicit backpressure mechanism.
- "How would you shrink the ID size, or extend the system's lifespan beyond the epoch's range?" → adjusting the bit allocation (fewer sequence/machine-ID bits if you have fewer machines or lower per-machine throughput) trades off maximum machines/throughput against a longer usable timestamp range before the 41-bit epoch counter wraps.
