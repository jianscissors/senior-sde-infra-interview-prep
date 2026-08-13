# Design a Chat System (e.g., WhatsApp/Slack)

## Clarifying questions to ask first
- 1:1 messaging only, or also group chats? What's the max group size?
- Delivery guarantees needed — at-least-once with client-side dedup, or stronger?
- Do we need read receipts, typing indicators, and online presence?
- Offline delivery required (message waiting when the recipient's device reconnects)?
- End-to-end encryption in scope, or is transport security (TLS) sufficient for this discussion?

## Requirements
**Functional**
- Send/receive messages 1:1 and in groups.
- Messages delivered even if the recipient is offline when sent.
- Message ordering preserved within a conversation.

**Non-functional**
- Low latency for online users (sub-second delivery).
- Durable — a message, once acknowledged to the sender, must not be lost.
- Scales to millions of concurrent persistent connections.

## Back-of-envelope estimation
- 50M concurrent connected users, each held open on a connection server → need many connection-server instances, each holding maybe 50-100K connections, so ~500-1000 connection servers just for holding sockets open.
- 500M messages/day (~5800/sec average, much higher at peak) — modest for a message broker, but the connection count is the real scaling challenge, not message throughput.

## High-level architecture
1. **Connection layer**: clients hold a persistent connection (WebSocket or long-poll) to a connection server; a client's connection server is tracked in a routing/presence service (which server holds which user's socket).
2. **Send path**: sender → their connection server → message service (persists message, determines recipient's connection server via the routing service) → if recipient online, push directly; if offline, store for later delivery (push notification to wake the client, or delivered on next connect).
3. **Group chat**: message service fans out to each group member's connection server (or a pub/sub topic per group that connection servers subscribe to on behalf of connected members).

## Deep dives

### Connection management at scale
- WebSocket (or gRPC bidi streaming) over long-poll for lower overhead and true push. Connection servers are stateful (they own live sockets) — this makes them harder to scale/deploy than stateless app servers, since a deploy/restart drops connections that must reconnect and re-subscribe. A routing table (e.g., in Redis) maps user ID → connection server, updated on connect/disconnect, so other services know where to push a message for an online user.

### Message ordering and delivery guarantees
- Order is only guaranteed to matter within a single conversation, not globally — assign each message a per-conversation monotonic sequence number (or a Snowflake-style time-ordered ID) so clients can detect gaps and re-request missing messages.
- At-least-once delivery is the realistic guarantee (network failures make exactly-once effectively impossible without idempotency) — the client dedups by message ID, and the server only clears a message from the "pending delivery" store after receiving an explicit ack from the recipient's client.

### Online presence
- Presence (online/offline/last-seen) is inherently approximate and eventually consistent — track it via connection open/close events plus a heartbeat with a short TTL (so a hard-crashed client without a clean disconnect still ages out of "online" quickly). Don't try to make presence strongly consistent across all of a user's contacts; broadcasting every presence flicker to everyone who might care doesn't scale — batch/throttle presence updates.

### Group chat fan-out
- For small-to-medium groups, fan out the message directly to each online member's connection server (similar to the 1:1 path, just N times). For very large groups (thousands of members, e.g., a broadcast channel), this starts to resemble the news-feed fan-out problem — consider a pub/sub topic per group that connection servers subscribe to only for currently-connected members, rather than the message service tracking every member individually.

### Offline delivery / push notifications
- If the recipient's connection server lookup finds no active connection, persist the message in a per-user "pending" store and trigger a push notification (APNs/FCM) to wake the client. On reconnect, the client syncs anything pending since its last-known sequence number per conversation.

## Key tradeoffs
- Stateful connection servers are operationally harder (connection draining on deploy, sticky routing) but are required for true low-latency push; a stateless long-poll design is simpler to operate but adds latency and server load from constant reconnects.
- Storing message history indefinitely vs. a retention window is a product/cost tradeoff — most infra interviews expect you to at least mention tiered storage (hot recent messages vs. cold archive) for the message history.

## Failure modes
- Connection server crashes → all its sockets drop; clients reconnect (with backoff/jitter to avoid a thundering herd) and get routed to a different server; the routing table must be updated promptly so senders don't push into a dead server.
- Routing/presence service down → fall back to "assume offline, use push notification path" so messages still get delivered, just with added latency, rather than being lost.

## Likely follow-ups
- "How do you handle a user with the app open on 3 devices simultaneously?" → the routing table maps user → set of connection-server/device entries, not a single one; fan out to all of the user's active sessions.
- "How would you add end-to-end encryption?" → the server becomes a blind relay for ciphertext; complicates group key management (each new/removed member requires re-keying) and server-side features that need plaintext (search, moderation) stop working without client-side workarounds.
