# Design a Web Crawler

## Clarifying questions to ask first
- Scale target — how many pages per day, and how fresh must the index be?
- Must we respect `robots.txt` and per-domain politeness (almost certainly yes)?
- Are we crawling the whole open web, or a bounded set of seed domains?
- Do we need to handle JavaScript-rendered pages (headless rendering), or is static HTML enough?

## Requirements
**Functional**
- Given seed URLs, discover and fetch linked pages, extract new links, and repeat.
- Avoid recrawling the same URL redundantly; avoid infinite loops.
- Respect per-site crawl policy (`robots.txt`, rate limits).

**Non-functional**
- Must scale horizontally to many parallel fetchers.
- Must not overwhelm any single target site (politeness is a hard constraint, not an optimization).
- Should prioritize freshness for high-value/frequently-changing pages.

## Back-of-envelope estimation
- 1B pages/month ≈ ~400 pages/sec average, but real crawls are bursty and rate-limited per domain far below that per site.
- URL frontier and dedup set at billions of URLs — a naive in-memory hash set doesn't fit on one machine; needs a distributed/sharded or probabilistic structure.

## High-level architecture
1. **URL frontier**: a prioritized, distributed queue of URLs to fetch, partitioned so fetches for the same domain are naturally grouped for politeness control.
2. **Fetcher workers**: pull a URL, check `robots.txt` (cached), fetch the page, respecting per-domain rate limits.
3. **Parser**: extract content and outbound links from the fetched page.
4. **Dedup/seen filter**: check extracted links against a "seen" set before enqueueing; skip already-crawled (or recently-crawled) URLs.
5. **Storage**: fetched content goes to a content store; scheduling metadata (last-crawled time, priority) goes to a separate metadata store.

## Deep dives

### URL frontier and dedup (bloom filter)
- The frontier is typically split into a **priority queue** (which URLs matter most to crawl next — based on page importance, update frequency, or explicit seed priority) feeding into **per-host sub-queues** so a scheduler can enforce politeness independently per domain.
- Exact dedup of billions of seen URLs is expensive to store and check with a plain hash set. A **bloom filter** gives fast, memory-efficient probabilistic membership testing — false positives (wrongly thinking a URL was already seen, so skipping it) are an acceptable tradeoff for a crawler, since missing an occasional URL is low-cost, whereas false negatives (which bloom filters never produce) would mean re-crawling forever.

### Politeness (per-domain rate limiting / robots.txt)
- Fetch and cache each domain's `robots.txt` before crawling it, honoring `Crawl-delay` and disallowed paths.
- Enforce a minimum delay between requests to the same host regardless of how many fetcher workers exist globally — this requires routing all URLs for a given host through a mechanism that tracks per-host last-fetch-time (e.g., consistent hashing of host → a specific queue/limiter shard), not just a global rate limit, or one popular domain could still get hammered by many workers acting independently.

### Distributed crawling coordination
- Partition the frontier by host (e.g., consistent hashing of domain) across worker shards so politeness state for a given host lives in one place and doesn't need cross-shard coordination for every fetch.
- Workers pull work from their shard's queue independently — no central scheduler needs to be in the hot path per-fetch, only for periodic rebalancing/priority updates.

### Freshness / recrawl scheduling
- Assign each URL an estimated change frequency (from observed history: how often has this page's content changed on previous crawls) and schedule recrawls proportionally — a news homepage might be recrawled hourly, a static reference page monthly. This is essentially an adaptive priority score fed back into the frontier's priority queue.

### Handling traps (infinite link chains)
- Set a max crawl depth from any seed, and detect degenerate patterns (e.g., calendar pages generating infinite date-incrementing URLs) via URL pattern heuristics (repeated path segments, absurdly long query strings) or a per-host budget cap so one pathological site can't consume unbounded crawler resources.

## Key tradeoffs
- Bloom filter dedup (memory-efficient, some false positives) vs. exact dedup (memory-expensive at this scale, no false positives) — bloom filters are the standard answer here because occasional missed re-crawls are cheap, unlike storage cost at billions of URLs.
- Depth-first crawling finds fewer distinct sites faster but risks getting stuck in one domain's link graph; breadth-first spreads coverage but takes longer to go deep on any one site — most crawlers use a priority-based hybrid rather than pure BFS/DFS.

## Failure modes
- A single fetcher worker crashes mid-fetch → the URL should be re-enqueued (with a lease/timeout on "in-progress" URLs, similar to a job queue) rather than silently lost.
- A target site starts returning errors or extreme latency → back off that host specifically (exponential backoff per-domain) rather than letting slow/broken hosts starve the throughput of fetcher workers that could be serving healthy hosts.

## Likely follow-ups
- "How would you detect and avoid crawling duplicate content at different URLs?" → content hashing (e.g., simhash for near-duplicate detection) after fetch, separate from URL-level dedup.
- "How do you prioritize which pages to crawl first from a massive frontier?" → a score combining page importance (e.g., link-graph-based, similar to PageRank), estimated change frequency, and freshness requirements, feeding the priority queue.
