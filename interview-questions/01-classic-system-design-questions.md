# Classic System Design Questions

General-purpose questions that show up in almost every system design loop, infra or not. Use these to practice the *process* (requirements → estimation → high-level design → deep dive → tradeoffs) before moving to the infra-specific set. For each, sketch: functional requirements, non-functional requirements (scale/latency/consistency), a high-level architecture, and 2-3 deep-dive areas.

1. **Design a URL shortener** (e.g., bit.ly)
   - Focus areas: key generation (counter+base62 vs hash vs random+collision check), read-heavy caching strategy, redirect latency, custom aliases, analytics/click tracking without slowing the redirect path, expiration.

2. **Design a rate limiter**
   - Focus areas: algorithms (token bucket, leaky bucket, fixed window, sliding window/log), where it lives (client, gateway, per-service sidecar), distributed rate limiting (shared counter in Redis vs approximate local counters), fairness across tenants.

3. **Design a news feed / timeline** (e.g., Twitter/Instagram feed)
   - Focus areas: fan-out-on-write vs fan-out-on-read (and hybrid for celebrity accounts), ranking, pagination/cursor design, cache invalidation on new posts, read amplification.

4. **Design a chat system** (e.g., WhatsApp/Slack)
   - Focus areas: connection management at scale (WebSocket/long-poll, connection servers), message ordering and delivery guarantees, online presence, group chat fan-out, offline delivery/push notifications, end-to-end encryption tradeoffs if asked.

5. **Design a web crawler**
   - Focus areas: URL frontier and dedup (bloom filter), politeness (per-domain rate limiting/robots.txt), distributed crawling coordination, freshness/recrawl scheduling, handling traps (infinite link chains).

6. **Design a notification system** (push/email/SMS fan-out)
   - Focus areas: multi-channel delivery, retry/backoff per channel, user preference/dedup rules, priority tiers, third-party provider rate limits and failover between providers.

7. **Design a ride-sharing / location-based matching system** (e.g., Uber dispatch)
   - Focus areas: geospatial indexing (geohash, quad-tree, S2), real-time location updates at scale, matching algorithm, handling hot geographic areas.

8. **Design a video streaming platform** (e.g., YouTube/Netflix)
   - Focus areas: upload/transcoding pipeline (async job processing), adaptive bitrate streaming, CDN strategy, metadata storage vs blob storage separation.

9. **Design a search autocomplete / typeahead system**
   - Focus areas: trie or prefix-index structure, ranking by frequency/personalization, latency budget (usually <100ms), how the index gets updated/rebuilt without downtime.

10. **Design a distributed unique ID generator** (e.g., Twitter Snowflake)
    - Focus areas: monotonicity requirements, clock drift handling, coordination-free generation (embed timestamp + machine ID + sequence in the ID) vs coordinated (centralized counter service).

## How to grade your own practice run
- Did you state assumptions and ask clarifying questions before diving in?
- Did you do at least one back-of-envelope calculation?
- Did you name the consistency model and justify it?
- Did you identify at least one bottleneck and one single point of failure, and address both?
- Did you discuss at least one explicit tradeoff rather than presenting the design as the only right answer?
