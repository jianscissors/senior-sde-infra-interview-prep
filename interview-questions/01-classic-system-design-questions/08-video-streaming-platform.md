# Design a Video Streaming Platform (e.g., YouTube/Netflix)

## Clarifying questions to ask first
- User-generated uploads (YouTube-style, unpredictable volume/format) or a curated catalog (Netflix-style, controlled ingestion)?
- Live streaming in scope, or video-on-demand only?
- Global audience — how important is playback start latency across regions?
- Do we need to design recommendation/search, or just storage, transcoding, and delivery?

## Requirements
**Functional**
- Users upload video; it becomes available for others to stream.
- Playback adapts to the viewer's network/device conditions.
- Metadata (title, description, view count) is queryable and searchable.

**Non-functional**
- Playback start latency should be low, and adaptive to avoid buffering.
- Storage and delivery must scale to massive object sizes and massive concurrent viewership (a viral video watched by millions simultaneously).
- Upload pipeline must handle large files reliably (resumable uploads).

## Back-of-envelope estimation
- A single hour of video at multiple bitrates/resolutions (transcoded output) can be several GB — multiply by upload volume, and storage is dominated by transcoded renditions, not the original upload.
- A viral video might be requested by millions of concurrent viewers — this must be served almost entirely from CDN edge caches, since no origin fleet is sized for that fan-out.

## High-level architecture
1. **Upload path**: client uploads (resumable, chunked) → raw video lands in blob storage → an async transcoding pipeline is triggered.
2. **Transcoding pipeline**: job queue processes the raw video into multiple resolutions/bitrates and packages it for adaptive streaming (e.g., HLS/DASH segments).
3. **Metadata service**: stores video metadata (title, owner, status, available renditions) separately from the actual video bytes.
4. **Playback path**: client requests a manifest (list of available renditions/segments) → fetches segments from the CDN, switching bitrate adaptively based on measured bandwidth.

## Deep dives

### Upload/transcoding pipeline (async job processing)
- Upload and transcoding must be decoupled — the user gets a fast "upload complete, processing" response while a background job pipeline (queue + worker pool) handles the CPU-heavy transcoding work, updating the video's status (processing → ready) when done. This is a standard async-job-processing pattern: durable job queue, idempotent workers, retry on failure, and a status callback/webhook or the metadata service reflecting progress.
- Large uploads should be chunked/resumable (so a dropped connection doesn't require restarting a multi-GB upload from zero) — typically via a multipart upload protocol against the blob store.

### Adaptive bitrate streaming
- The transcoding pipeline produces multiple renditions (e.g., 240p/480p/720p/1080p/4K, each at appropriate bitrates) segmented into short chunks (a few seconds each), described by a manifest file (HLS `.m3u8` or DASH `.mpd`).
- The client player continuously measures its actual download throughput and switches to a higher- or lower-bitrate rendition for the *next* segment accordingly — this is entirely client-driven logic; the server's job is just to have all renditions available as independently fetchable segments.

### CDN strategy
- Nearly all playback traffic should be served from CDN edge caches, not the origin — video segments are immutable once produced, making them ideal for aggressive, long-TTL caching. The origin (blob storage + transcoding output) is only hit on a cache miss (first request for that segment from a given edge region).
- For predictable high-demand content (e.g., a scheduled major release), pre-warming the CDN ahead of the traffic spike avoids a thundering-herd of cache misses all hitting the origin simultaneously at launch.

### Metadata storage vs. blob storage separation
- Video bytes (large, immutable once transcoded, accessed via CDN) belong in object storage; metadata (title, description, view counts, comments, processing status — small, frequently updated/queried, needs indexing for search) belongs in a database designed for that access pattern. Conflating the two (e.g., storing metadata inside the blob store) makes both simple queries and cache-friendly video delivery harder.
- View counts specifically are extremely high-write, low-consistency-requirement data — batch/approximate counting (increment a local counter, periodically flush aggregated counts) rather than a synchronous strongly-consistent counter per view avoids that becoming a bottleneck.

## Key tradeoffs
- Transcoding every uploaded video into all renditions immediately (fast playback start for everyone, high upfront compute cost including for videos nobody watches) vs. transcoding on-demand/lazily for rarely-requested renditions (saves compute, adds latency on the first request for an unusual rendition) — most large-scale systems transcode the common renditions eagerly and can afford to be lazier about rare combinations (e.g., an unusual device codec).
- Strong consistency isn't needed almost anywhere in this system (view counts, even "is this video ready yet" status) — eventual consistency with a good UX around the transitional states (e.g., "processing" spinner) is the standard, much cheaper approach.

## Failure modes
- A transcoding worker crashes mid-job → the job must be resumable/retryable (idempotent job design, checkpointing or simply restarting the whole transcode job cleanly) rather than leaving the video stuck in "processing" forever.
- CDN edge node or region has an outage → clients should fail over to another edge/region transparently (this is typically handled by the CDN provider's routing, but worth naming as a requirement of your CDN choice).

## Likely follow-ups
- "How would you support live streaming instead of (or in addition to) VOD?" → live changes the pipeline fundamentally — segments are produced and must be available within seconds of being captured (low-latency HLS/DASH or WebRTC for sub-second use cases), and there's no "process once, serve forever" — every segment is transient.
- "How do you avoid re-transcoding the same content if two users upload the same video?" → content-based deduplication (hash the raw file) before triggering a fresh transcoding job, reusing existing renditions when a byte-identical (or perceptually identical, via fingerprinting) upload is detected.
