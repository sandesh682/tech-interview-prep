---
tags: [system-design, phase-6, case-studies, practice, real-world]
status: not-started
phase: 6
---

# Phase 6 — Real-World System Design Cases

> [!IMPORTANT]
> **This is where preparation becomes skill.** Reading about system design is not enough — you must practice designing these systems end-to-end, out loud, under time pressure. Aim to complete all 10 cases at least once before your interview.

---

## 🧭 Overview

This phase applies everything from Phases 1–5 to real-world systems. Each case follows the RESHADED framework: Requirements → Estimation → System Interface → High-Level Design → Architecture Deep-Dive → Data Model → Edge Cases → Deep Dive.

Study each system by: first attempting to design it yourself (timed, 45 min), then checking your design against the notes here and resources linked.

---

## 📋 Cases Index

| # | System | Core Challenge | Difficulty | Status |
|---|--------|---------------|------------|--------|
| 1 | [[#URL Shortener]] | Redirection, encoding, analytics | Beginner | ⬜ |
| 2 | [[#Social Media Feed (Twitter/Instagram)]] | Fan-out, feed generation, scale | Medium | ⬜ |
| 3 | [[#Chat System (WhatsApp/Slack)]] | Real-time delivery, storage, groups | Medium | ⬜ |
| 4 | [[#Notification System]] | Multi-channel, reliability, at-scale | Medium | ⬜ |
| 5 | [[#Ride-Hailing (Uber/Ola)]] | Geo-search, matching, real-time | Hard | ⬜ |
| 6 | [[#Video Streaming (Netflix/YouTube)]] | CDN, encoding, recommendation | Hard | ⬜ |
| 7 | [[#Payment Gateway (Razorpay/PayU)]] | Idempotency, consistency, security | Hard | ⬜ |
| 8 | [[#Search Autocomplete / Typeahead]] | Trie, ranking, latency | Medium | ⬜ |
| 9 | [[#Distributed Rate Limiter]] | Algorithms, Redis, multi-region | Medium | ⬜ |
| 10 | [[#E-Commerce Platform (Amazon/Flipkart)]] | Catalog, cart, orders, inventory | Hard | ⬜ |

---

## 📖 Case Studies

---

### URL Shortener

**Systems like:** bit.ly, TinyURL

**Key Requirements:**
- Given a long URL → generate a short code (e.g., `myapp.io/xK3p9`)
- Short URL redirects to original URL
- Analytics: click count, geography (optional)

**Estimation:**
- 100M URLs created/day → ~1,200 writes/s
- 10:1 read:write ratio → ~12,000 reads/s
- Average URL = ~100 bytes → 100M × 100B = 10 GB/day storage

**Core Design Decisions:**

*Encoding strategy:*
- **Hash (MD5/SHA256)**: Hash the URL, take first 7 characters. Problem: collision possible.
- **Base62 encoding of auto-increment ID**: `[0-9][a-z][A-Z]` → 62^7 = ~3.5 trillion unique codes. Simple and collision-free.
- **Pre-generated short codes**: Generate and store a pool of unique codes in advance.

*Redirection:*
- HTTP 301 (Permanent Redirect) — browser caches → less load on your server, but no analytics on repeat visits
- HTTP 302 (Temporary Redirect) — no caching → every click hits your server → better analytics

*Storage:* Key-value store (DynamoDB or Redis) — `shortCode → longUrl`. No relations needed, O(1) lookup.

*Read scaling:* Cache short codes in Redis (most URLs are accessed in bursts after creation, then rarely). Cache TTL: ~24 hours.

**Data Model:**
```
urls: {
  short_code: string (PK),
  long_url: string,
  created_at: timestamp,
  user_id: string (nullable),
  click_count: number
}
```

**Edge Cases:**
- Custom short codes (e.g., `myapp.io/my-blog`) — check uniqueness
- Expiry (TTL on the record)
- Malicious URL detection (integrate Safe Browsing API)

---

### Social Media Feed (Twitter/Instagram)

**Key Requirements:**
- Post tweets/photos
- Follow other users
- See a timeline of posts from people you follow (sorted by time or relevance)

**Core Challenge: Fan-out**

When a user with 10M followers posts, you need to deliver that post to 10M feeds.

**Fan-out on Write (Push model):**
- On post creation → push post to every follower's feed cache
- Pros: Feed reads are O(1) — just read pre-computed feed
- Cons: A celebrity post triggers 10M writes simultaneously (hot spot)

**Fan-out on Read (Pull model):**
- On feed load → query all followees' recent posts and merge
- Pros: No write amplification
- Cons: Feed load is expensive (query N timelines and merge)

**Hybrid approach (Twitter's actual approach):**
- Regular users → fan-out on write (push)
- Celebrities (>1M followers) → fan-out on read (pull at read time, merge with pre-computed feed)

**Storage:**
- Posts → PostgreSQL or Cassandra (with `user_id + timestamp` as compound key for efficient timeline reads)
- Feed cache → Redis sorted sets (`ZADD feed:{userId} timestamp postId`)
- Media → S3 + CDN

**Edge Cases:**
- User unfollows → remove their posts from your feed cache (lazy: just filter at read time)
- Delete post → remove from all feeds (expensive) — lazy deletion with tombstones
- Algorithm vs chronological feed — recommendation system needed for algorithmic

---

### Chat System (WhatsApp/Slack)

**Key Requirements:**
- 1-on-1 messaging
- Group messaging (up to 500 members)
- Online presence / last seen
- Message delivery receipts (sent ✓, delivered ✓✓, read ✓✓ in blue)

**Real-time delivery:** WebSockets (persistent connection per client to a Chat Server)

**Architecture:**
- **Chat Servers**: Maintain WebSocket connections. Stateful per connection.
- **Presence Service**: Tracks online users (Redis with TTL — heartbeat resets TTL)
- **Message Storage**: Cassandra (`(channel_id, message_id) PK`) — write-heavy, time-ordered reads

**Message flow (1-on-1):**
1. Alice sends message → Chat Server A
2. Chat Server A → Kafka (persists message)
3. Consumer → stores in Cassandra
4. Consumer checks if Bob is online (Presence Service)
5. If online → push to Bob's Chat Server via pub/sub (Redis pub/sub or Kafka)
6. Bob's Chat Server → WebSocket → Bob's client

**Group messages:**
- Fan-out to all group members' message queues
- For large groups (1000+): fan-out on read (members pull from the group's message store)

**Message ID strategy:** Use a monotonically increasing ID per channel (Snowflake ID or DB sequence) for ordering.

**Edge Cases:**
- Offline users → messages stored; delivered when they come online
- End-to-end encryption → keys managed client-side; server stores ciphertext
- Message search → Elasticsearch index on message content

---

### Notification System

**Key Requirements:**
- Push notifications (iOS/Android via APNs/FCM)
- Email notifications (SendGrid/SES)
- SMS notifications (Twilio)
- In-app notifications
- User preferences (opt-out per channel)

**Architecture:**
1. **Notification Service** — receives trigger events from other services
2. **Event Queue** (Kafka) — decouple triggering from sending
3. **Worker Services** — one per channel (Push Worker, Email Worker, SMS Worker)
4. **Third-party providers** — APNs, FCM, SendGrid, Twilio
5. **Notification Log** — store all sent notifications (for history and debugging)

**Reliability:**
- Workers must be idempotent (retry on failure without duplicate sends)
- Track delivery status per notification per channel
- Dead-letter queue for failed notifications

**User Preferences:**
- Lookup user's notification preferences before sending
- Cache preferences in Redis to avoid DB call per notification

**Rate limiting:** Don't bombard users. Token bucket per user across all channels.

---

### Ride-Hailing (Uber/Ola)

**Key Requirements:**
- User requests a ride (location)
- Match to nearby available driver
- Real-time location tracking
- Price estimation
- Trip management

**Core Challenge: Geo-Search**
Find all available drivers within X km of a user in real-time.

**Geo-indexing approaches:**
- **Geohash**: Encode lat/long as a string. Adjacent geohashes are nearby. Query nearby geohashes in DB.
- **QuadTree**: Recursive spatial partitioning. Each cell subdivides when > N drivers.
- **Redis GEOADD/GEORADIUS**: Built-in geo commands — simplest for interview

**Driver location updates:**
- Driver app sends GPS coordinates every ~4 seconds
- Location goes to a **Location Service** → stores in Redis (fast, ephemeral)
- Historical trip tracking → Kafka → eventually persisted in DB/S3

**Matching Service:**
1. User requests ride at location L
2. Matching Service queries Location Service for drivers within 2km
3. Rank drivers by distance + rating + ETA
4. Send ride request to top driver (timeout 10s, then next driver)

**Dynamic Pricing (Surge):**
- Supply/demand ratio per geohash cell
- If demand > supply × threshold → multiply base price by surge factor
- Surge computed periodically (every 5 min) by a pricing service

---

### Video Streaming (Netflix/YouTube)

**Key Requirements:**
- Upload videos
- Process and transcode to multiple resolutions
- Stream to users with low buffering

**Upload pipeline:**
1. Client uploads raw video to S3 (via pre-signed URL)
2. Upload complete event → Kafka
3. **Transcoding Workers** (auto-scaling): Convert to HLS/DASH, multiple resolutions (4K, 1080p, 720p, 480p, 360p)
4. Processed segments stored in S3 → invalidated/pushed to CDN
5. Metadata (title, thumbnail, duration) stored in PostgreSQL

**Adaptive Bitrate Streaming (ABR):**
- Video divided into small segments (2–10 seconds)
- Player selects segment resolution based on available bandwidth
- **HLS (HTTP Live Streaming)**: Apple format, widely supported
- **DASH (Dynamic Adaptive Streaming over HTTP)**: Industry standard

**CDN Strategy:**
- Static segments served from CDN edge nodes
- CDN pre-warms popular content (new episode release → push to edges)

**Recommendation System:**
- Collaborative filtering (users similar to you liked X)
- Content-based filtering (similar to what you've watched)
- In interviews: mention ML service consumes viewing events from Kafka

---

### Payment Gateway (Razorpay/PayU)

**Key Requirements:**
- Accept payments (credit card, UPI, net banking, wallets)
- High reliability (payment can't be lost)
- Idempotency (no double charges)
- Security (PCI-DSS compliance)

**Core Design:**

*Idempotency:* Every payment request includes an `idempotency_key`. Server stores key → result. Retries return cached result.

*Payment flow:*
1. Merchant's server calls Payment Gateway API
2. Gateway creates a `payment_order` in PENDING state
3. Gateway calls bank/UPI/card network (external call — can fail)
4. Bank responds → Gateway updates payment to SUCCESS or FAILED
5. Webhook sent to merchant (with retry logic)

*Reliability:*
- Write payment attempt to DB before external call
- On external call success → update DB atomically
- Outbox pattern for webhook delivery
- Separate reconciliation job to catch discrepancies between gateway and bank records

*Security:*
- Tokenize card data — store `card_token` not raw card numbers (via a PCI-compliant vault)
- TLS everywhere
- Rate limit per merchant API key

---

### Search Autocomplete / Typeahead

**Key Requirements:**
- As user types, suggest completions (e.g., "iph" → "iphone 15", "iphone case", "iphone charger")
- Low latency (< 100ms)
- Ranked by popularity

**Core Data Structure: Trie**
- Each node represents a character
- Each leaf stores top-K completions for that prefix (pre-computed)
- Search: traverse trie for prefix → O(prefix_length)

**At scale:**
- Can't fit the full trie in memory for all possible prefixes
- Pre-compute top-K suggestions per prefix in batch (offline job)
- Store in Redis hash: `prefix → [suggestion1, suggestion2, ... suggestionK]`
- CDN/browser caching for common prefixes

**Data collection:**
- Log every search query
- Aggregate (Kafka → Spark → frequency table)
- Rebuild trie/Redis daily or weekly

**Filtering:** Remove profanity, competitor brand names.

---

### Distributed Rate Limiter

**Key Requirements:**
- Limit API calls per user/IP (e.g., 1000 req/min per API key)
- Consistent across multiple servers
- Low latency (can't add 50ms to every request)

**Single-server approach:** In-memory counter per key — fast but doesn't work across multiple app servers.

**Distributed approach with Redis:**
- **Fixed Window**: `INCR key`, `EXPIRE key 60` (if new key)
- **Sliding Window Log**: `ZADD key timestamp`, `ZREMRANGEBYSCORE` to remove old entries, `ZCARD` to count
- **Token Bucket in Redis**: Lua script to atomically check and decrement tokens

**Lua script atomicity:** Redis executes Lua scripts atomically. Use for multi-step operations like token bucket.

**Multi-region rate limiting:** Each region has its own Redis. Accept slight over-counting (local counters per region) rather than sync across regions.

---

### E-Commerce Platform (Amazon/Flipkart)

**Key Requirements:**
- Product catalog (browse, search)
- Shopping cart
- Checkout / Order placement
- Inventory management
- Order tracking

**Services:**
- **Catalog Service** → PostgreSQL + Elasticsearch for search
- **Cart Service** → Redis (fast, ephemeral; sync to DB on checkout)
- **Order Service** → PostgreSQL (ACID for transactions)
- **Inventory Service** → PostgreSQL with optimistic locking (prevent overselling)
- **Payment Service** → See [[#Payment Gateway]]
- **Notification Service** → Order confirmation emails/SMS

**Inventory consistency:**
- Pessimistic locking (`SELECT FOR UPDATE`) during checkout — guarantees no oversell but reduces throughput
- Optimistic locking — check version on update, retry on conflict — higher throughput, some retries

**Flash sale challenge:**
- Inventory count pre-loaded into Redis
- `DECR inventory:{productId}` — atomic decrement
- If result < 0 → reject (sold out)
- Orders queued in Kafka → processed by Order Service asynchronously
- Prevents DB from being overwhelmed by concurrent checkouts

---

## 📚 Suggested Resources

| Resource | Why |
|----------|-----|
| *System Design Interview Vol 1 & 2* — Alex Xu | Cases 1, 2, 3, 8 aligned exactly |
| ByteByteGo — YouTube playlist | Visual walkthroughs of most cases above |
| Gaurav Sen — "Design Uber" (YouTube) | Best geo-matching explanation |
| High Scalability — highscalability.com | Real architecture posts from companies |
| Swiggy, Razorpay, PhonePe engineering blogs | India-specific product designs |

---

## ✅ Progress Tracker

- [ ] URL Shortener — designed end-to-end (timed)
- [ ] Social Media Feed — understand fan-out on write vs read
- [ ] Chat System — WebSocket flow, message ordering, delivery receipts
- [ ] Notification System — multi-channel, idempotency, retry
- [ ] Ride-Hailing — geohash/QuadTree, driver matching
- [ ] Video Streaming — transcoding pipeline, HLS/DASH, CDN
- [ ] Payment Gateway — idempotency key, reconciliation, security
- [ ] Search Autocomplete — trie, Redis, ranking by popularity
- [ ] Distributed Rate Limiter — Redis Lua script, multi-region
- [ ] E-Commerce — inventory consistency, flash sale handling

---

*Next → [[Phase 7 - Advanced & Specialized Topics]]*
