# Design a Search Autocomplete / Typeahead System

## Clarifying questions to ask first
- Personalized suggestions (per-user history) or global popularity-based only?
- How fresh must suggestions be — can the index rebuild on a delay (minutes/hours), or does it need near-real-time updates (trending topics)?
- What's the target latency budget? (Typically very tight, since this fires on every keystroke.)
- Scope of the corpus — a fixed dictionary of terms, or derived from live query logs?

## Requirements
**Functional**
- Given a prefix, return the top-K most relevant completions.
- Suggestions ranked by relevance (typically frequency/popularity, possibly personalized).

**Non-functional**
- Very low latency (usually targeting well under 100ms end-to-end, since this fires on every keystroke and must feel instantaneous).
- High read QPS (every keystroke of every active user is a request) with a much lower write/update rate.
- Index must be updatable without taking the system down (query patterns shift over time, e.g., trending terms).

## Back-of-envelope estimation
- A user typing a 10-character query can trigger up to 10 autocomplete requests — read volume is a multiple of actual search volume.
- The suggestion corpus (distinct terms/phrases worth indexing) is typically much smaller than the full query log — often pruned to the top N million terms by frequency, since long-tail rare queries aren't worth indexing for prefix lookup.

## High-level architecture
1. **Offline/async aggregation**: query logs are periodically aggregated (batch job) into term frequencies, building/refreshing the prefix index.
2. **Serving layer**: a read-optimized prefix index (trie or equivalent) held largely in memory, sharded and replicated across serving nodes.
3. **Query path**: client sends the current prefix on each keystroke (often debounced) → serving layer returns top-K completions from the index, ranked by precomputed score.

## Deep dives

### Trie or prefix-index structure
- A **trie** naturally supports prefix lookup: walking down the trie by the input prefix's characters lands you at the subtree of all completions, and if each node caches its top-K most frequent completions beneath it, retrieval is just reading that cached list — no need to traverse and re-rank the whole subtree per query.
- At large scale, a plain in-memory trie for the full corpus may not fit on one machine — shard it (e.g., by first character or a hash of the prefix) across multiple serving nodes, or use a more compact structure (e.g., a sorted array of terms with binary search for the prefix range, which is more memory-efficient than a pointer-heavy trie at the cost of slightly more complex updates).

### Ranking by frequency/personalization
- Base ranking is typically term frequency from aggregated query logs — precomputed offline, since recomputing "top-K per prefix" live on every request would be far too slow.
- Personalization (boosting a user's own history/context) is usually layered on top of the base global ranking at serving time — merge a small personalized candidate set (e.g., user's own recent searches matching the prefix) with the global top-K, rather than maintaining a full personalized index per user, which wouldn't scale.

### Latency budget (usually <100ms)
- This tight budget is why the index lives in memory, not a general-purpose database — a database round-trip with disk I/O and query planning overhead is not fast enough for this pattern repeated on every keystroke.
- Client-side debouncing (wait for a short pause in typing before firing the request) and request cancellation (abandon an in-flight request if the user has already typed further) reduce load without needing server-side changes — worth mentioning as a complementary optimization, not a substitute for a fast server-side index.

### How the index gets updated/rebuilt without downtime
- Rebuild the index offline (new trie/structure built from the latest aggregated frequencies) and swap it in atomically once complete — serving nodes either load the new index and atomically switch a pointer to it, or a rolling deploy brings up nodes with the new index and drains traffic from nodes on the old one. Never mutate the live serving structure in place under read traffic.
- For faster-moving signals (e.g., a suddenly trending term), a lightweight incremental overlay (a small, frequently-refreshed structure checked in addition to the main index) can surface fresh terms faster than waiting for the next full rebuild cycle.

## Key tradeoffs
- A full trie is intuitive and fast for prefix walks but memory-heavy at scale; a sorted-array-plus-binary-search (or a compressed trie variant) is more memory-efficient but slightly more complex to update incrementally — most systems bias toward memory efficiency at scale.
- Precomputing top-K per node (fast reads, some storage/build overhead) vs. computing top-K per query (no precompute cost, but too slow for this latency budget) — precomputation is essentially mandatory given the latency target.

## Failure modes
- A serving shard goes down → its portion of the prefix space becomes unavailable; replicate each shard so a single node failure doesn't blank out autocomplete for an entire range of prefixes.
- Offline aggregation job fails or lags → the index goes stale but keeps serving its last-known-good version rather than falling back to no suggestions — staleness degrades quality gracefully, unlike an outage.

## Likely follow-ups
- "How would you filter out offensive or low-quality suggestions?" → a blocklist/filter step applied either at aggregation time (exclude from the index entirely) or at serving time (filter the candidate list before returning) — usually both, since new terms can spike between aggregation cycles.
- "How would you support autocomplete across multiple languages?" → typically separate indexes per language/locale (different tokenization rules, different corpora), selected based on the user's locale rather than one merged global index.
