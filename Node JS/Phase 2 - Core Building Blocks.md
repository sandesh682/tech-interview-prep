---
tags: [system-design, phase-2, caching, queues, databases, rate-limiting]
status: not-started
phase: 2
---

# Phase 2 — Core Building Blocks

> [!IMPORTANT]
> This is the most **frequently tested phase** in system design interviews. Every real-world design — URL shortener, Twitter, Uber — uses these components. Master them deeply before moving on.

---

## 🧭 Overview

This phase covers the individual components that appear in nearly every system design answer: caches, message queues, databases, rate limiters, and API gateways. You need to know not just *what* they are, but *when* to use them, *how* they fail, and *what trade-offs* they introduce. Think of these as your "lego bricks."

---

## 📋 Topics Table

| Topic | Level | Why It Matters | Est. Time |
|-------|-------|----------------|-----------|
| Caching — strategies, layers, eviction | Must-Know | Every scalable system uses caching | 3 hrs |
| Redis — data structures, use cases | Must-Know | De-facto caching + pub/sub layer | 2 hrs |
| Message Queues — Kafka, RabbitMQ, SQS | Must-Know | Async processing, decoupling | 3 hrs |
| Relational Databases — indexing, ACID | Must-Know | Foundational for data modeling | 2 hrs |
| NoSQL Databases — types and use cases | Must-Know | MongoDB is your turf — go deeper | 2 hrs |
| Rate Limiting — algorithms | Must-Know | API protection, fairness | 2 hrs |
| API Gateway | Must-Know | Entry point for microservices | 1 hr |
| Service Discovery | Must-Know | How microservices find each other | 1.5 hrs |
| Blob / Object Storage (S3-like) | Must-Know | Files, images, videos | 1 hr |
| Search — Elasticsearch basics | Good-to-Know | Search features in any product | 1.5 hrs |
| Distributed Locks | Good-to-Know | Prevents race conditions at scale | 1 hr |

---

## 📖 Detailed Notes

### Caching

Caching stores the result of expensive computations or DB queries in fast memory so future requests don't need to recompute or re-query.

**Caching Layers (outermost to innermost):**
1. **Client-side** — Browser cache, service worker cache
2. **CDN** — Edge caching for static and semi-static content
3. **Load Balancer / Reverse Proxy** — Nginx/Varnish can cache responses
4. **Application-level** — In-memory cache in your Node.js app (risky — not shared)
5. **Distributed Cache** — Redis, Memcached (shared across app servers)
6. **Database query cache** — DB-level caching (generally deprecated in favor of Redis)

**Cache Strategies:**

| Strategy | How It Works | Best For |
|----------|-------------|----------|
| Cache-Aside (Lazy Loading) | App checks cache; on miss, loads from DB and populates cache | Read-heavy, unpredictable access |
| Write-Through | Write to cache and DB simultaneously | Strong consistency, write-heavy |
| Write-Behind (Write-Back) | Write to cache; async write to DB | Performance-critical writes |
| Read-Through | Cache automatically fetches from DB on miss | Simplifies app logic |
| Refresh-Ahead | Pre-emptively refresh cache before expiry | Low-latency, predictable access patterns |

**Cache Eviction Policies:**
- **LRU (Least Recently Used)** — Most common, evicts the item not accessed for the longest time
- **LFU (Least Frequently Used)** — Evicts the item accessed least often
- **FIFO** — Evicts oldest entry regardless of access pattern
- **TTL (Time To Live)** — Item expires after a fixed time

**Cache Problems to Know:**
- **Cache Stampede / Thundering Herd**: Many requests hit DB simultaneously when a popular cache entry expires → Solution: mutex lock on cache miss, or staggered TTLs
- **Cache Penetration**: Requests for keys that don't exist in DB or cache (e.g., user IDs that don't exist) → Solution: Bloom filter or cache null results
- **Cache Avalanche**: Many cache entries expire at the same time → Solution: Jitter (randomize TTLs)

> [!TIP] You likely know this — just revise
> You've used Redis in Node.js. For interviews, be specific: "I'd use Cache-Aside with a 5-minute TTL and jitter to prevent cache avalanche, backed by Redis with LRU eviction."

---

### Redis

Redis is an in-memory data structure store. It's used for much more than caching:

| Use Case | Redis Feature |
|----------|--------------|
| Caching | Key-Value with TTL |
| Session storage | Key-Value (fast reads) |
| Rate limiting | Counters + TTL, or Sorted Sets |
| Leaderboards | Sorted Sets (ZADD, ZRANK) |
| Pub/Sub messaging | PUBLISH / SUBSCRIBE |
| Distributed locks | SETNX + TTL (or Redlock algorithm) |
| Job queues | Lists (LPUSH / BRPOP) |
| Real-time analytics | HyperLogLog (cardinality estimation) |

> [!NOTE]
> Redis is **single-threaded** for command execution. This means no locking needed for atomic operations, but CPU-bound operations can block. Use Redis Cluster for horizontal scaling.

---

### Message Queues

Message queues decouple producers from consumers, enabling async processing and absorbing traffic spikes.

**Key Concepts:**
- **Producer** — publishes messages to a queue/topic
- **Consumer** — reads and processes messages
- **Broker** — the queue system itself (Kafka, RabbitMQ, SQS)
- **At-least-once delivery** — message may be delivered more than once → consumers must be idempotent
- **At-most-once delivery** — message delivered once or not at all (fire and forget)
- **Exactly-once delivery** — hardest, requires transactional producers/consumers (Kafka supports this)

**Kafka vs RabbitMQ vs SQS:**

| | Kafka | RabbitMQ | AWS SQS |
|--|-------|----------|---------|
| Model | Log-based (append only) | Traditional queue | Managed cloud queue |
| Replay | ✅ (consumers can re-read) | ❌ | ❌ (limited) |
| Ordering | Per-partition | Per-queue | FIFO queues only |
| Throughput | Very high (millions/s) | High | High |
| Use case | Event streaming, analytics | Task queues, RPC | Simple decoupling on AWS |
| Retention | Configurable (days/forever) | Until consumed | 14 days max |

> [!TIP] You likely know this — just revise
> You've integrated with AWS services. Know that Kafka is preferred when you need **event replay** (audit log, event sourcing) and SQS/RabbitMQ when you just need simple **task decoupling**.

**When to use a message queue in design:**
- Sending emails/notifications after an action (async)
- Processing payments (don't block the API response)
- Fan-out to multiple services (order placed → inventory, billing, notification)
- Absorbing write spikes (queue fills up, workers drain at their own pace)

---

### Relational Databases — Key Concepts

> [!TIP] You likely know this — just revise
> Focus on the concepts below that go beyond basic CRUD.

**ACID Properties:**
- **Atomicity** — transaction succeeds or fails completely (no partial writes)
- **Consistency** — DB moves from one valid state to another
- **Isolation** — concurrent transactions don't interfere
- **Durability** — committed data survives crashes (WAL / write-ahead log)

**Indexing:**
- **B-Tree Index** — Default for most SQL DBs. Good for range queries and equality.
- **Hash Index** — Only for exact equality. Not range queries.
- **Composite Index** — Multi-column. Column order matters (`(user_id, created_at)` helps queries filtering by user_id then sorting by time).
- **Covering Index** — Index includes all columns in the query → no table lookup needed.

> [!WARNING]
> Indexes speed up reads but **slow down writes** (index must be updated on every insert/update). Don't over-index.

**When to use SQL:** Financial transactions, user data, anything requiring complex joins, strong consistency.

---

### NoSQL Databases

| Type | Examples | Use Case |
|------|----------|----------|
| Document | MongoDB, Firestore | User profiles, product catalogs, flexible schema |
| Key-Value | Redis, DynamoDB | Session storage, shopping carts, leaderboards |
| Column-family | Cassandra, HBase | Time-series, analytics, write-heavy at scale |
| Graph | Neo4j, Amazon Neptune | Social graphs, recommendation engines, fraud detection |

> [!TIP] You likely know this — just revise
> You know MongoDB well. For interviews: explain **when you'd choose MongoDB over PostgreSQL**. Answer: when the schema is flexible/evolving, when you need to store nested/hierarchical data naturally, or when you need horizontal sharding from day one.

**MongoDB-specific interview points:**
- Documents are BSON (binary JSON)
- Schema-less but you should use validation in production
- Aggregation pipeline replaces SQL's GROUP BY / JOIN
- Atlas Search for full-text search on top of MongoDB

---

### Rate Limiting

Rate limiting protects APIs from abuse, DoS attacks, and runaway clients.

**Algorithms:**

| Algorithm | How It Works | Pros | Cons |
|-----------|-------------|------|------|
| Fixed Window Counter | Count requests per window (e.g., 100/min) | Simple | Burst at window boundary |
| Sliding Window Log | Store timestamp of each request, count in last N seconds | Accurate | Memory-heavy |
| Sliding Window Counter | Approximate using weighted counter from prev window | Good accuracy + memory | Slightly complex |
| Token Bucket | Tokens added at fixed rate; each request consumes a token | Allows bursts | Token management complexity |
| Leaky Bucket | Requests processed at fixed rate; overflow is dropped | Smooth output rate | Can delay legitimate bursts |

**Implementation with Redis:**
- `INCR key` + `EXPIRE` for fixed window
- `ZADD` + `ZCOUNT` for sliding window log
- Atomic Lua scripts for multi-step operations

**Where to apply:** API Gateway, Nginx, application middleware, or a dedicated rate-limiting service.

---

### API Gateway

An API Gateway is a reverse proxy with superpowers:
- **Authentication & Authorization** (JWT validation, OAuth)
- **Rate Limiting**
- **Request routing** (route `/users/*` to user-service, `/orders/*` to order-service)
- **Request/Response transformation**
- **SSL termination**
- **Logging & monitoring**
- **Circuit breaking**

Examples: AWS API Gateway, Kong, Nginx + Lua, Traefik

> [!TIP] You likely know this — just revise
> You've used AWS API Gateway. For interviews, position the API Gateway as the **single entry point** for your microservices, and note that it introduces a single point of failure — mitigate with horizontal scaling and redundancy.

---

### Service Discovery

In a microservices environment, services start/stop dynamically. Service discovery lets services find each other without hardcoded IPs.

**Client-side discovery:** The client queries a **service registry** (e.g., Consul, Eureka) and load-balances itself.
**Server-side discovery:** The client hits a load balancer which queries the registry.

**DNS-based discovery:** Services register with DNS. Simpler but less real-time.

In Kubernetes (which you've used), service discovery is built-in via **kube-dns** — services are addressable by name (e.g., `http://user-service:3000`).

---

### Blob / Object Storage

For files, images, videos — don't store in a database. Use object storage:
- **AWS S3** — de-facto standard. Infinitely scalable, 11 9s durability.
- Store metadata (filename, owner, size, URL) in your DB
- Generate **pre-signed URLs** for secure, time-limited direct access
- Use CDN in front of S3 for fast delivery

> [!TIP] You likely know this — just revise
> You've used AWS. Know the pattern: client → your API → generate pre-signed S3 URL → client uploads directly to S3 (avoids routing large files through your server).

---

### Distributed Locks

When multiple instances of your service try to update the same resource, you need a distributed lock.

**Redis SETNX + TTL approach:**
```
SET lock_key unique_id NX PX 30000
```
- `NX` = only set if not exists
- `PX 30000` = expire in 30 seconds (prevents deadlock if process crashes)
- Use the unique_id to ensure only the lock holder can release it

**Redlock Algorithm** — Multi-node Redis lock for higher reliability (acquire lock on majority of N Redis nodes).

Use cases: Payment processing, inventory reservation, leader election.

---

## 📚 Suggested Resources

| Resource | Why |
|----------|-----|
| ByteByteGo — "Top Caching Strategies" | Best visual explainer |
| Gaurav Sen — "Message Queues" playlist | Kafka + RabbitMQ explained intuitively |
| Redis Documentation — redis.io/docs | Data types reference |
| *DDIA* Ch. 3 (Storage Engines) | Deep dive on indexes |
| Martin Fowler — "CQRS" & "Saga" articles | Patterns that use queues |

---

## ✅ Progress Tracker

- [ ] Caching layers and all 5 strategies
- [ ] Cache problems: stampede, penetration, avalanche — and solutions
- [ ] Redis data structures and use cases beyond caching
- [ ] Kafka vs RabbitMQ vs SQS — choose the right one
- [ ] Message delivery guarantees (at-least-once, exactly-once)
- [ ] SQL ACID properties and indexing types
- [ ] NoSQL types and when to use each
- [ ] Rate limiting algorithms — token bucket and sliding window
- [ ] API Gateway responsibilities and trade-offs
- [ ] Service discovery mechanisms
- [ ] S3 pattern for file uploads (pre-signed URLs)
- [ ] Distributed locks with Redis

---

*Next → [[Phase 2.5 - LLD & SOLID Principles]]* (before moving to data storage, solidify LLD foundations)
