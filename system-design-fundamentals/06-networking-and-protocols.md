# Networking & Protocols

## Why it comes up
Infra roles sit closer to the network than most backend roles — expect questions on how requests actually travel, TLS overhead, and protocol tradeoffs (TCP vs UDP, HTTP/2 vs HTTP/3, gRPC vs REST).

## TCP vs UDP
- **TCP:** connection-oriented, reliable (retransmits), ordered, flow/congestion controlled. Overhead of handshake (3-way) and head-of-line blocking within a connection. Default for anything needing reliability (HTTP, gRPC, databases).
- **UDP:** connectionless, no reliability/ordering guarantees, lower overhead/latency. Used for DNS, video/voice streaming, QUIC (built on UDP), gaming — anywhere you'd rather drop a stale packet than wait for retransmission.

## DNS
- Resolution chain: client → recursive resolver → root → TLD → authoritative nameserver.
- Caching via TTL at every layer; low TTLs enable faster failover but increase query load.
- Used for load balancing/failover at a coarse grain (GSLB) — round robin, weighted, latency-based, geo-based records.
- DNS is a common hidden bottleneck/failure point in outages — don't forget it exists in your design.

## TLS/HTTPS
- Handshake: TCP handshake, then TLS handshake (cert exchange, key exchange — TLS 1.3 reduced this to 1-RTT, down from 2-RTT in TLS 1.2).
- TLS termination: often done at the load balancer/edge (offloads CPU cost from app servers); "TLS passthrough" when the backend must see the original connection (e.g., mTLS to the origin).
- mTLS (mutual TLS): both sides present certs — standard for service-to-service auth inside a service mesh.
- Session resumption / TLS session tickets reduce handshake cost for repeat connections.

## HTTP evolution
- **HTTP/1.1:** text-based, one request per connection at a time (head-of-line blocking), keep-alive reuses connections but still serial per connection (pipelining largely unused).
- **HTTP/2:** binary framing, multiplexed streams over a single TCP connection (no app-level head-of-line blocking), header compression (HPACK), server push (rarely used now). Still has TCP-level head-of-line blocking — one lost packet stalls all streams on that connection.
- **HTTP/3 (QUIC):** runs over UDP, each stream has independent loss recovery so one lost packet doesn't stall other streams, faster connection establishment (0-RTT resumption), built-in TLS 1.3. Growing default for latency-sensitive edge traffic.

## RPC & API styles
- **REST:** resource-oriented, HTTP verbs, human-readable (JSON), widely interoperable, weaker typing, chattier for fine-grained operations.
- **gRPC:** HTTP/2-based, Protobuf (binary, strongly typed, smaller payloads, schema evolution via field numbers), supports streaming (client/server/bidi), codegen for clients — the default choice for internal service-to-service infra communication at scale.
- **GraphQL:** client specifies exactly the fields it needs, avoids over/under-fetching, adds complexity (N+1 query problem, caching is harder than REST's URL-based caching).
- **Webhooks:** server-initiated callback over HTTP — simple async integration pattern, needs retry/signature-verification handling on the receiving side.

## Connection-level concepts
- **Keep-alive / connection pooling:** reusing TCP+TLS connections avoids repeated handshake cost — critical at scale (client-side connection pools to backends/DBs).
- **Timeouts at every hop:** connect timeout, read timeout, request timeout — must be set explicitly and tuned tighter than the caller above you (to avoid resource pileup while waiting on a hung downstream).
- **Retries with backoff + jitter:** exponential backoff prevents synchronized retry storms; jitter further desynchronizes clients; always cap retry count and combine with circuit breakers.
- **Circuit breakers:** stop calling a failing downstream after an error-rate threshold, fail fast, periodically probe to recover (half-open state) — prevents cascading failure.

## Common follow-ups
- "Why would you choose gRPC over REST for a given interface?" → internal, high-throughput, strongly-typed contract, need for streaming → gRPC; public-facing, broad client compatibility, human debuggability → REST.
- "How does HTTP/2 multiplexing help under high fan-out?" → many logical requests share one TCP connection, avoiding per-request handshake and browser connection-limit issues.
- "What happens to in-flight requests when you terminate TLS at the LB and the LB restarts?" → connection draining, health-check-aware deploys, client-side retry logic.
