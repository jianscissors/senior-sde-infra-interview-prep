# Design a News Feed / Timeline (e.g., Twitter/Instagram Feed)

## Clarifying questions to ask first
- Read-heavy or write-heavy skew — average follower count vs. celebrity accounts with millions of followers?
- Does the feed need to be strictly chronological, or ranked (relevance/engagement-based)?
- Real-time delivery expectations (seconds) or is "eventually shows up" acceptable?
- Do we need to support pagination/infinite scroll with stable ordering as new posts arrive?

## Requirements
**Functional**
- User posts content; followers see it in their feed.
- Feed supports pagination (scroll back through history).
- Optional: ranking/relevance, not just recency.

**Non-functional**
- Feed load latency should be low (sub-second) for a good UX.
- Highly read-heavy: feed reads vastly outnumber posts written.
- Must tolerate highly skewed fan-out (a celebrity's post fans out to millions instantly).

## Back-of-envelope estimation
- 200M daily active users, each loading their feed ~5x/day → 1B feed reads/day (~12K reads/sec average, higher at peak).
- 50M posts/day (~580 writes/sec average).
- Average follower count ~200, but power-law tail: some accounts have 50M+ followers — this skew is the crux of the design.

## High-level architecture
1. **Write path**: user posts → write to post store → fan-out service decides how to propagate to followers' feeds.
2. **Read path**: user requests feed → read pre-computed feed (fan-out-on-write) or merge posts from followed accounts at request time (fan-out-on-read) → paginate and return.

## Deep dives

### Fan-out-on-write vs fan-out-on-read
- **Fan-out-on-write (push)**: on post creation, immediately write the post ID into every follower's precomputed feed list (e.g., a Redis list per user). Feed reads become a cheap single lookup. Great for typical users, but a celebrity with 50M followers turns one post into 50M writes — both slow and wasteful if most followers never open the app that day.
- **Fan-out-on-read (pull)**: feed reads merge posts from all followed accounts at request time (fan-in query across N accounts, sorted by time). No write amplification, but read latency grows with the number of accounts followed and becomes expensive at scale for active readers.
- **Hybrid (the real-world answer)**: fan-out-on-write for the vast majority of normal accounts, fan-out-on-read (or a hybrid merge) for celebrity/high-follower accounts — the feed read for a user merges their precomputed feed with a live pull of any celebrity accounts they follow. This is what Twitter's actual architecture does.

### Ranking
- Chronological is simplest but not what most production feeds ship — ranking uses a scoring model (recency, engagement prediction, affinity to poster) computed either at write time (approximate, cached) or at read time (freshest signal, more expensive). A common pattern: pre-filter a candidate set (e.g., last 1000 posts from followed accounts) cheaply, then run a more expensive ranking model only on that candidate set rather than the full corpus.

### Pagination / cursor design
- Offset-based pagination breaks under concurrent inserts (items shift, causing duplicates/skips). Use cursor-based pagination keyed on a stable, monotonically ordered value (e.g., post timestamp + post ID as a tiebreaker, or a Snowflake-style ID that's inherently time-ordered) so "give me everything older than this cursor" is stable regardless of new inserts.

### Cache invalidation on new posts
- Precomputed feeds (Redis lists) are appended to, not fully recomputed, on each new post — O(1) per follower rather than a full feed rebuild. Cap the cached feed length (e.g., last 1000 items) and fall back to the read-time path/database for anyone paging back further than that.

## Key tradeoffs
- Fan-out-on-write trades write amplification for read speed; fan-out-on-read trades read latency/complexity for cheap writes. The hybrid approach is strictly better in practice but adds real implementation complexity (two code paths to maintain and keep consistent).
- Strong consistency (every follower sees the post in the same order instantly) is not worth the cost here — feeds are a canonical eventual-consistency use case; a few seconds of propagation delay is imperceptible to users.

## Failure modes
- Fan-out worker queue backs up (e.g., after a celebrity post) → feed delivery lags; degrade gracefully by prioritizing normal accounts' fan-out and processing celebrity fan-out (or falling back to pull-based merge) asynchronously without blocking the whole pipeline.
- Precomputed feed store (Redis) down → fall back to a live fan-out-on-read query against the post database at reduced performance rather than failing the feed entirely.

## Likely follow-ups
- "How do you handle a user who follows 10,000 accounts?" → for the pull side of the hybrid, this is where fan-out-on-read gets expensive — consider capping the live-merge account count or using an approximate/sampled merge for extreme outliers.
- "How would you delete a post and have it disappear from everyone's feed?" → tombstone the post ID; feed reads filter out tombstoned IDs; precomputed lists don't need immediate physical removal since they're rebuilt/trimmed over time.
