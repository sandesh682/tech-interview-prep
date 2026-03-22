---
tags: [system-design, phase-3, databases, sharding, replication, sql, nosql]
status: not-started
phase: 3
---

# Phase 3 — Data & Storage Deep Dive

> [!IMPORTANT]
> Data modeling decisions are **permanent** (or very costly to change). Interviewers pay close attention to how you choose and structure storage. With your MongoDB background, you're ahead — but go deeper on replication, sharding, and SQL trade-offs.

---

## 🧭 Overview

Every system is ultimately defined by its data: how it's stored, how it scales, and how it stays consistent under load. This phase goes deeper than Phase 2's overview — you'll learn how databases scale horizontally, how replication works, and how to make smart storage choices for different workload types.

---

## 📋 Topics Table

| Topic | Level | Why It Matters | Est. Time |
|-------|-------|----------------|-----------|
| SQL vs NoSQL decision framework | Must-Know | First question in every data design | 1.5 hrs |
| Database replication | Must-Know | High availability, read scaling | 2 hrs |
| Database sharding (partitioning) | Must-Know | Write scaling, large datasets | 2.5 hrs |
| Indexing deep dive | Must-Know | Query performance | 1.5 hrs |
| Database normalization vs denormalization | Must-Know | Schema design trade-offs | 1 hr |
| OLTP vs OLAP | Must-Know | Choosing the right DB for the workload | 1 hr |
| Time-series databases | Good-to-Know | Metrics, IoT, monitoring systems | 1 hr |
| Search engines (Elasticsearch) | Good-to-Know | Full-text search in products | 1.5 hrs |
| Data warehouses & lakes | Good-to-Know | Analytics, reporting | 1 hr |
| Connection pooling | Must-Know | DB bottleneck in Node.js services | 1 hr |

---

## 📖 Detailed Notes

### SQL vs NoSQL Decision Framework

> [!TIP]
> Don't say "I'll use MongoDB because I know it." Say "I'm choosing X because the access patterns, schema evolution needs, and consistency requirements point to it."

**Choose SQL when:**
- Data is relational and joins are needed (users → orders → products)
- ACID transactions are critical (payments, banking)
- Schema is stable and well-understood upfront
- Reporting and complex queries are needed
- Examples: PostgreSQL (best default choice), MySQL

**Choose NoSQL when:**
- Schema is flexible or evolving rapidly (product catalog with varying attributes)
- Need horizontal write scaling from the start
- Access patterns are simple (get-by-key, get-by-user)
- Data is hierarchical/nested naturally (JSON documents)
- Need extreme read/write throughput (Cassandra for time-series)

**The default advice for interviews:** Start with PostgreSQL. Justify NoSQL only when there's a clear reason.

---

### Database Replication

Replication = keeping copies of data on multiple servers for **availability** and **read scaling**.

**Primary-Replica (Master-Slave) Replication:**
- All **writes** go to the **primary**
- **Replicas** receive changes (via replication log) and serve **reads**
- If primary fails → promote a replica (failover), brief downtime possible

**Multi-Primary (Multi-Master) Replication:**
- Multiple nodes accept writes
- Conflict resolution needed (who wins if two nodes update the same row simultaneously?)
- Complex, used for geo-distributed setups where low write latency in each region is critical

**Replication Lag:**
- Replicas may be slightly behind the primary (asynchronous replication)
- Users might read stale data after a write (eventual consistency)
- Solution: "Read your own writes" — route a user's reads to primary right after their write

**Synchronous vs Asynchronous Replication:**

| | Synchronous | Asynchronous |
|--|-------------|--------------|
| Durability | Higher (replica confirms before ACK) | Lower (replica may lag) |
| Write latency | Higher | Lower |
| Use case | Financial systems | Read scaling, analytics replicas |

> [!NOTE]
> In MongoDB, replication is handled by **Replica Sets** — primary + secondaries. Reads can be configured with `readPreference: secondary` for read scaling (with potential staleness). This is exactly the same concept, just MongoDB's implementation.

---

### Database Sharding (Partitioning)

Sharding = splitting data **horizontally** across multiple DB instances (shards) so each shard holds a subset of the data.

**When to shard:** When a single DB node can't handle the write throughput or storage requirements.

**Sharding Strategies:**

| Strategy | How | Pros | Cons |
|----------|-----|------|------|
| **Range-based** | Shard by range (e.g., user_id 0–1M on shard 1) | Simple, range queries efficient | Hot spots if data is skewed |
| **Hash-based** | `shard = hash(key) % num_shards` | Even distribution | Range queries span all shards |
| **Directory-based** | Lookup table maps key → shard | Flexible, can relocate shards | Lookup table is a single point of failure |
| **Geo-based** | Shard by geography | Low latency for regional data | Uneven data if regions differ in size |

**Problems with sharding:**

- **Hot shards**: One shard gets disproportionate traffic (e.g., a celebrity's posts on a social platform)
  - Solution: Add a random suffix to the shard key, or use virtual nodes
- **Cross-shard joins**: JOIN across two tables on different shards is expensive
  - Solution: Denormalize data; avoid joins across shards
- **Resharding**: Adding or removing shards requires data redistribution — **Consistent Hashing** minimizes the number of keys that need to move

**Consistent Hashing:**
- Shards and keys are mapped to a ring
- Each key is assigned to the nearest shard clockwise on the ring
- Adding a shard only redistributes keys from one neighboring shard (not all)
- Used by Cassandra, DynamoDB, Redis Cluster

> [!TIP]
> In MongoDB Atlas and MongoDB sharded clusters, sharding is built-in. Knowing the underlying concepts (consistent hashing, hot shards, shard key selection) shows senior-level depth.

---

### Indexing Deep Dive

Indexes are data structures that speed up reads at the cost of write overhead and storage.

**B-Tree (Default in PostgreSQL, MySQL, MongoDB):**
- Balanced tree structure
- O(log n) reads
- Supports equality and range queries
- Order of columns in composite index matters!

**Composite Index Rules:**
- `INDEX(a, b, c)` helps queries filtering on `a`, `a + b`, or `a + b + c`
- Does NOT help a query filtering only on `b` or `c`
- Rule: "Left-prefix rule"

**Covering Index:**
- If all columns in a query are in the index, the DB doesn't need to read the actual row
- Massive performance win for read-heavy queries

**Index on cardinality:**
- High cardinality (user_id, email) → index is very effective
- Low cardinality (boolean, status with 3 values) → index often not used by query planner

**MongoDB-specific:**
- Compound indexes follow the same left-prefix rule
- `explain()` shows whether your query is using an index (IXSCAN vs COLLSCAN)
- Sparse indexes — only index documents where the field exists

> [!WARNING]
> Every index adds overhead to INSERT, UPDATE, DELETE. In a write-heavy system (e.g., logging), too many indexes will kill write performance.

---

### Normalization vs Denormalization

**Normalization:** Remove redundancy. Split data into related tables. Data lives in one place.
- Pros: No update anomalies, smaller storage
- Cons: Joins are expensive at scale

**Denormalization:** Duplicate data to avoid joins. Store redundant information together.
- Pros: Faster reads (no joins), simpler queries
- Cons: Risk of data inconsistency, more storage

> [!NOTE]
> In practice, most production systems are partially denormalized. The principle: **normalize first, denormalize where profiling shows joins are a bottleneck**.

In MongoDB, denormalization is the default — embedding related data in the same document (e.g., embedding an order's items inside the order document rather than referencing them separately).

---

### OLTP vs OLAP

| | OLTP | OLAP |
|--|------|------|
| Purpose | Day-to-day transactions | Analytics and reporting |
| Query type | Simple, point queries (get user by ID) | Complex, aggregations over large data |
| Data volume per query | Small | Large |
| Latency requirement | Low (ms) | Can be high (seconds) |
| Optimized for | Writes | Reads |
| Examples | PostgreSQL, MySQL, MongoDB | Redshift, BigQuery, Snowflake, ClickHouse |

**Column-store vs Row-store:**
- OLTP: Row-store (reads/writes entire rows)
- OLAP: Column-store (reads only the columns needed for aggregation — much faster for analytics)

---

### Time-Series Databases

For data that is inherently time-indexed (metrics, logs, sensor readings):

- **InfluxDB** — popular for monitoring and IoT
- **TimescaleDB** — PostgreSQL extension for time-series
- **Prometheus** — metrics + alerting (with Grafana for visualization)
- **Amazon Timestream** — managed AWS offering

**Key features:** Automatic data downsampling (keep high-res recent data, compress old data), time-range queries, retention policies.

---

### Connection Pooling

> [!TIP] You likely know this — just revise
> Every Node.js app needs connection pooling to avoid creating a new DB connection per request.

- DB connections are expensive to create (TCP + auth + session overhead)
- A connection pool maintains a set of reusable connections
- In Node.js: Mongoose's `poolSize`, `pg-pool`, or `Sequelize`'s `pool` config
- Set pool size based on: `(num_cores × 2) + effective_spindle_count` (PgBouncer recommendation)
- If pool is exhausted → requests queue up → latency spikes

**Connection Pooler (PgBouncer):** Sits between app and PostgreSQL. Multiplexes many app connections into fewer DB connections. Essential for serverless/high-concurrency apps.

---

### Elasticsearch for Search

Elasticsearch is a distributed search and analytics engine built on Apache Lucene.

**Key concepts:**
- **Index** ≈ a table (but schema is flexible)
- **Document** ≈ a row (stored as JSON)
- **Inverted Index** — maps words to the documents containing them (what makes full-text search fast)
- **Relevance scoring** — results sorted by relevance, not just existence

**Use cases in system design:**
- Full-text product search
- Log aggregation and search (ELK stack: Elasticsearch + Logstash + Kibana)
- Autocomplete / typeahead

**Sync pattern:** Write to primary DB (PostgreSQL/MongoDB) → publish change event → consumer updates Elasticsearch. ES is a **read replica** for search, not the source of truth.

---

## 📚 Suggested Resources

| Resource | Why |
|----------|-----|
| *DDIA* Ch. 3, 5, 6 | Storage engines, replication, partitioning — essential reading |
| Gaurav Sen — "Database Sharding" (YouTube) | Intuitive consistent hashing explanation |
| ByteByteGo — "SQL vs NoSQL" (YouTube) | Decision framework explained visually |
| Use The Index, Luke — use-the-index-luke.com | Deep dive on SQL indexing |
| MongoDB University — M201 (Performance) | Free, covers MongoDB indexing in depth |

---

## ✅ Progress Tracker

- [ ] SQL vs NoSQL decision framework — can justify choice for 3 different scenarios
- [ ] Primary-Replica replication — sync vs async, replication lag
- [ ] Multi-Primary replication and conflict resolution
- [ ] Hash-based vs Range-based sharding + hot spot problem
- [ ] Consistent hashing — understand the ring model
- [ ] Composite index left-prefix rule
- [ ] Covering indexes and when they matter
- [ ] Normalization vs denormalization — know when to denormalize
- [ ] OLTP vs OLAP — different DBs for different workloads
- [ ] Connection pooling in Node.js/production apps
- [ ] Elasticsearch inverted index + sync pattern

---

*Next → [[Phase 4 - Distributed Systems Concepts]]*
