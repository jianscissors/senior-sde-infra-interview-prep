# Design a Notification System (Push/Email/SMS Fan-out)

## Clarifying questions to ask first
- Which channels are in scope — push, email, SMS, in-app? Multiple simultaneously per notification?
- Are notifications triggered by internal events (e.g., "user X liked your post") or scheduled/bulk (e.g., marketing blast to 10M users)?
- Do users have channel preferences (opt out of email but allow push) and quiet hours?
- What's the latency requirement — near-real-time for transactional notifications (e.g., 2FA code) vs. best-effort for marketing?

## Requirements
**Functional**
- Accept a notification request (from internal services or a scheduler) and deliver it via one or more channels.
- Respect user preferences (opted-in channels, do-not-disturb windows).
- Retry failed deliveries; avoid duplicate delivery of the same notification.

**Non-functional**
- High throughput for bulk sends (millions of notifications in a short window) without falling behind.
- Time-sensitive notifications (e.g., OTP codes) need low latency even during a bulk-send burst — they can't queue behind a marketing blast.
- Must survive individual third-party provider (APNs, SendGrid, Twilio) outages without losing notifications.

## Back-of-envelope estimation
- A single marketing campaign to 20M users across push+email could mean 40M send requests dispatched in a short window — this bursty, bulk-fan-out load is the main scaling challenge, distinct from steady-state transactional volume.

## High-level architecture
1. **Ingestion API**: internal services (or a campaign scheduler) submit notification requests.
2. **Preference/dedup filter**: check user's channel preferences and dedup rules before proceeding.
3. **Priority queues per channel**: separate queues (or priority tiers within a queue) so high-priority transactional notifications aren't stuck behind bulk marketing sends.
4. **Channel-specific worker pools**: consume from queues and call out to the relevant third-party provider (APNs/FCM for push, SendGrid/SES for email, Twilio for SMS).
5. **Retry/DLQ**: failed sends retry with backoff; permanently-failed messages land in a dead-letter queue for inspection.

## Deep dives

### Multi-channel delivery
- Treat each channel as an independent pipeline with its own queue, worker pool, and provider integration — a spike or outage in email delivery shouldn't back up push delivery. A single notification request can fan out to multiple channel-specific messages that proceed independently.

### Retry/backoff per channel
- Providers fail differently: transient (5xx, timeout — retry with exponential backoff and jitter) vs. permanent (invalid device token, bounced email address — don't retry, mark the destination as bad and stop future sends to it). Cap total retry attempts and duration so a permanently-undeliverable notification doesn't retry forever.

### User preference / dedup rules
- Preferences (which channels a user allows, quiet hours, digest vs. immediate) should be checked once at ingestion, not duplicated per channel worker, to keep the logic in one place and avoid drift.
- Dedup: if the same logical event fires a notification twice (e.g., a retried upstream call), dedup on an idempotency key (event ID + user ID + channel) with a short TTL so the same notification isn't delivered twice to the same channel.

### Priority tiers
- Separate queues (or a priority field consumed preferentially) for transactional (OTP, security alerts — deliver in seconds) vs. bulk/marketing (best-effort, can take minutes to fully drain). Worker pools for the high-priority tier should be provisioned/scaled independently so a marketing blast can't starve them by consuming all available provider-connection capacity.

### Third-party provider rate limits and failover
- Each provider imposes its own rate limits (e.g., APNs connection limits, SendGrid sends/sec) — the worker pool must throttle to stay under these, using the same rate-limiter patterns (token bucket) as any outbound integration.
- For channels with multiple viable providers (e.g., SMS via Twilio or a backup vendor), support failover: if the primary provider's error rate spikes, route new sends to a secondary provider rather than queuing indefinitely behind a degraded one.

## Key tradeoffs
- Strict per-user ordering of notifications (e.g., don't show notification B before A) adds complexity (partitioned queues keyed by user) that most systems don't need — most notification systems don't guarantee cross-notification ordering, only best-effort timeliness.
- At-least-once delivery (possible duplicates, mitigated by idempotency keys) is much simpler to build than exactly-once, and is the standard choice — user-visible duplicate notifications are an acceptable, rare cost.

## Failure modes
- A provider (e.g., APNs) goes down → that channel's queue backs up; workers should detect the elevated error rate and back off (circuit breaker) rather than burning through retry budget hammering a dead provider, while other channels continue unaffected.
- Bulk campaign floods the ingestion API → queue absorbs the burst; if queue depth grows unbounded, apply backpressure to the campaign scheduler (slow down acceptance) rather than letting the queue grow until workers fall permanently behind.

## Likely follow-ups
- "How would you prevent a single misconfigured campaign from spamming 20M users repeatedly?" → rate-limit sends per campaign ID, and require an idempotency/campaign-run key so a retried campaign trigger doesn't resend to everyone.
- "How do you know a push notification was actually seen, not just sent?" → that's a separate delivery-receipt/read-receipt pipeline (client acks on receipt/open), fed back asynchronously — don't conflate "handed to provider" with "delivered to device" in your success metrics.
