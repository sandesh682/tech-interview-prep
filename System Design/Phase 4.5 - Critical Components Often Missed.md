---
tags: [system-design, phase-4.5, unique-id, bloom-filter, gossip, feature-flags, disaster-recovery]
status: not-started
phase: 4.5
---

# Phase 4.5 — Critical Components Often Missed

> [!IMPORTANT]
> These topics are tested frequently at product-based companies but often absent from standard prep guides. Each one is a small, self-contained concept that can be asked as a standalone question ("Design a unique ID generator") or appear as a sub-problem inside a larger design.

---

## 🧭 Overview

This phase covers the "connective tissue" of distributed system design: the utility components and concepts that keep coming up but don't fit neatly into a single phase. Topics here range from unique ID generation (comes up in almost every HLD) to disaster recovery planning (comes up in every senior SRE/senior SDE discussion).

---

## 📋 Topics Table

| Topic | Level | Why It Matters | Est. Time |
|-------|-------|----------------|-----------|
| Unique ID Generation (Snowflake, UUID, etc.) | Must-Know | Required in nearly every system design | 1.5 hrs |
| Bloom Filters | Must-Know | Space-efficient membership testing | 1 hr |
| Consistent Hashing (deep dive) | Must-Know | Sharding + load balancing | 1.5 hrs |
| Gossip Protocol | Good-to-Know | Failure detection in distributed clusters | 1 hr |
| Disaster Recovery — RTO, RPO | Must-Know | Senior SDE must know availability planning | 1 hr |
| Feature Flags / Feature Toggles | Must-Know | Safe rollouts, A/B testing, kill switches | 1 hr |
| Back-Pressure & Flow Control | Must-Know | Prevents cascading failures in pipelines | 1 hr |
| Merkle Trees | Good-to-Know | Data integrity in distributed systems | 1 hr |
| Schema Migration (zero-downtime) | Must-Know | Real production concern with SQL | 1.5 hrs |
| Chaos Engineering | Good-to-Know | Production resilience validation | 45 min |
| Data Consistency Patterns (read-repair, hinted handoff) | Must-Know | How NoSQL repairs inconsistencies | 1 hr |

---

## 📖 Detailed Notes

---

### Unique ID Generation

Almost every system needs globally unique IDs for records. The requirements:
- Globally unique (no collisions across services/machines)
- Sortable by time (for pagination, ordering)
- High throughput (thousands per second)
- Short enough to fit in a database column

**Options:**

#### UUID v4 (random)
- Format: `550e8400-e29b-41d4-a716-446655440000` (128 bits, 36 chars)
- Pros: Dead simple, no coordination needed
- Cons: Not sortable by time, large index size, not human-friendly

#### UUID v7 (time-ordered — modern choice)
- Time-ordered UUIDs — first bits are a timestamp
- Sortable + unique — best of both worlds for most use cases
- Supported natively in PostgreSQL 17+

#### Auto-increment (DB sequence)
- Pros: Simple, sortable, compact integers
- Cons: Single point of failure (DB), reveals business metrics (order #1000 shows scale), hard to shard

#### Twitter Snowflake ID
The most common interview answer for unique ID generation at scale.

**Structure (64 bits total):**
```
| 41 bits timestamp (ms since epoch) | 10 bits machine ID | 12 bits sequence |
```
- **41 bits timestamp**: ~69 years of milliseconds
- **10 bits machine ID**: Up to 1024 machines (can split: 5 bits datacenter + 5 bits machine)
- **12 bits sequence**: 4096 IDs per millisecond per machine

**Properties:**
- ~4096 × 1024 = ~4 million IDs/ms globally
- Naturally sortable by time (timestamp is the most significant bits)
- No coordination between machines (each machine has its own ID)
- IDs are 64-bit integers — compact and index-friendly

> [!NOTE]
> Snowflake IDs are used by Twitter, Instagram, Discord, and many others. In interviews, if asked "how would you generate unique IDs?", Snowflake is the gold-standard answer for large-scale systems.

**Clock skew problem:** If a machine's clock goes backward (NTP adjustment), it could generate duplicate IDs. Solution: Wait until the clock catches up, or use a sequence that guards against this.

#### Ticket Server (Flickr approach)
Centralized auto-increment server. All services ask this server for the next ID.
- Pros: Simple, sequential
- Cons: Single point of failure — mitigate with two ticket servers (odd/even IDs)

---

### Bloom Filters

A Bloom filter is a **space-efficient probabilistic data structure** that tells you: "This element is **definitely not** in the set, OR it is **probably** in the set."

**Properties:**
- No false negatives — if Bloom filter says "not in set", it's 100% correct
- Possible false positives — if it says "in set", it might be wrong (probability depends on size/hash functions)
- Cannot delete elements (standard Bloom filter)
- Space: ~10 bits per element for ~1% false positive rate

**How it works:**
1. A bit array of M bits, all initialized to 0
2. K hash functions
3. **Add element**: Hash element with all K functions → set those K bits to 1
4. **Check element**: Hash with all K functions → if ALL K bits are 1 → probably present. If ANY bit is 0 → definitely not present.

**Use cases in system design:**
- **Cache penetration defense**: Before querying DB, check Bloom filter — if key definitely doesn't exist, skip DB query entirely
- **Username availability**: Check if username is taken without querying DB for every check
- **Cassandra/HBase**: Used internally to avoid reading SSTables that don't contain the key
- **Chrome Safe Browsing**: URLs checked against a Bloom filter locally before sending to Google's server
- **Distributed DB**: Check if a node has a row before sending a cross-node query

> [!TIP]
> Mentioning Bloom filters when discussing cache penetration (Phase 2) shows real depth. In interviews: "To prevent cache penetration attacks, I'd use a Bloom filter — if the filter says the key definitely doesn't exist, we skip the DB lookup entirely."

---

### Consistent Hashing — Deep Dive

(Introduced in [[Phase 3 - Data & Storage Deep Dive]] — expanded here)

**The problem with naive hashing:** `server = hash(key) % N`. If N changes (add/remove server), almost all keys map to a different server → massive cache misses or data movement.

**Consistent hashing solution:**
- Map both servers and keys onto a **circular ring** (0 to 2³²-1)
- A key is assigned to the first server clockwise from its hash position
- Adding a server: only keys between the new server and its predecessor need to move
- Removing a server: only keys on that server move to its clockwise neighbor

**Virtual Nodes (vnodes):**
Problem: With few real servers, distribution can be uneven.
Solution: Each physical server has multiple **virtual nodes** spread around the ring (e.g., Server A has virtual nodes at positions 100, 300, 450, 700...).
- Result: More even key distribution
- Benefit: When a server is added/removed, load is spread across multiple servers, not just neighbors

**Used by:** Cassandra, DynamoDB, Amazon's Dynamo, Redis Cluster, Couchbase.

---

### Gossip Protocol

A **gossip protocol** (also called epidemic protocol) is a peer-to-peer communication mechanism where nodes periodically share their state with a random subset of other nodes.

**How it works:**
1. Every T seconds, each node picks K random peers
2. Exchanges state information (node health, ring membership, data versions)
3. Information spreads exponentially — reaches all N nodes in O(log N) rounds

**Use cases:**
- **Failure detection**: Nodes detect when another node has gone silent (Cassandra uses gossip for this)
- **Cluster membership**: New nodes announce themselves; departing nodes propagate their exit
- **Data version propagation**: Spreading version vectors across nodes

**Properties:**
- Decentralized — no single point of failure
- Eventually consistent propagation
- Scales to large clusters (unlike broadcasting to all nodes)
- Tolerates network partitions

**Used by:** Cassandra (for ring membership and failure detection), Riak, DynamoDB.

> [!NOTE]
> You don't need to implement gossip — just know: "In Cassandra, failure detection is handled by gossip protocol — nodes exchange state with random peers every second, and a node is marked as down if it hasn't been heard from in a configurable timeout."

---

### Disaster Recovery — RTO & RPO

Two critical metrics for business continuity planning:

**RPO — Recovery Point Objective:**
How much **data loss** is acceptable? The maximum age of data that must be recovered.
- RPO = 0: No data loss (synchronous replication, very expensive)
- RPO = 1 hour: You can afford to lose up to 1 hour of data (asynchronous replication every hour)

**RTO — Recovery Time Objective:**
How long can the system be **down**? The maximum time to restore service.
- RTO = 0: No downtime (active-active, most expensive)
- RTO = 4 hours: System can be offline up to 4 hours before business is critically impacted

**DR Strategies (cost increases with reliability):**

| Strategy | RTO | RPO | How |
|----------|-----|-----|-----|
| Backup & Restore | Hours–Days | Hours | Periodic backups to S3; restore on failure |
| Pilot Light | Minutes–Hours | Minutes | Minimal replica running; scale up on disaster |
| Warm Standby | Minutes | Seconds–Minutes | Scaled-down copy, always running |
| Multi-site Active-Active | Near-zero | Near-zero | Full replica in another region, always serving traffic |

> [!TIP]
> In interviews, when discussing availability: "The business requires 99.99% uptime, which means ~52 minutes downtime/year. That points to a Warm Standby or Active-Active DR strategy. The RPO should be < 5 minutes, so we'd need synchronous replication for the primary DB or at least frequent async replication."

**Runbook:** A documented step-by-step procedure for responding to specific incidents. Having runbooks means faster RTO (less time figuring out what to do).

---

### Feature Flags (Feature Toggles)

Feature flags let you **decouple code deployment from feature release**. Code ships to production but the feature is hidden behind a flag.

**Types:**
| Type | Purpose | Lifespan |
|------|---------|---------|
| **Release toggle** | Hide incomplete features in prod | Short (days–weeks) |
| **Experiment toggle** | A/B testing — 50% users see new UI | Medium (weeks) |
| **Ops toggle** | Kill switch for expensive features (e.g., turn off recommendations under load) | Long-term |
| **Permission toggle** | Enable feature for specific users/tenants (beta users, premium plan) | Long-term |

**Implementation:**
- Simple: Environment variables (no runtime update)
- Better: Config in Redis/DB — change flag without deploying
- Best: Dedicated service (LaunchDarkly, Unleash, AWS AppConfig) — targeting rules, gradual rollouts, analytics

**Canary release with feature flags:**
1. Deploy new code with flag OFF
2. Enable flag for 1% of users
3. Monitor error rate, latency, business metrics
4. If healthy → increase to 10% → 50% → 100%
5. If issues → flip flag OFF instantly (no rollback needed)

> [!TIP] You likely know this — just revise
> You've done CI/CD with GitHub Actions. Feature flags are the next level — they let you deploy every day safely by decoupling deployment from release.

---

### Back-Pressure & Flow Control

**The problem:** A fast producer overwhelms a slow consumer. Without back-pressure, the consumer runs out of memory or crashes.

**Back-pressure:** The mechanism by which a slow consumer signals to the producer to slow down.

**Strategies:**

| Strategy | Mechanism | Trade-off |
|----------|-----------|-----------|
| **Drop** | Drop messages when buffer full | Data loss but system stable |
| **Throttle** | Signal producer to slow down | Producer slows, increased latency |
| **Buffer** | Queue overflow in memory/disk | Delayed processing, eventual consistency |
| **Block** | Producer blocks until consumer is ready | Simple, but can deadlock |

**In Node.js streams:**
```javascript
// Readable stream with back-pressure
const readStream = fs.createReadStream('largefile.csv');
const writeStream = fs.createWriteStream('output.csv');
readStream.pipe(writeStream);  // pipe() handles back-pressure automatically
```

**In Kafka:** Consumers pull from topics (not pushed to). If a consumer is slow, it just reads more slowly — the broker doesn't overwhelm it. The offset tracks progress.

**In system design:** When your ingestion service receives a traffic spike, the message queue absorbs it (acts as a buffer). Workers process at their own pace — this IS back-pressure in action.

---

### Merkle Trees

A **Merkle tree** is a hash tree where every leaf node contains the hash of data, and every non-leaf node contains the hash of its children.

```
            Root Hash
           /          \
     Hash(A+B)      Hash(C+D)
     /      \        /      \
  Hash(A) Hash(B) Hash(C) Hash(D)
    |        |       |       |
  Data A  Data B  Data C  Data D
```

**Properties:**
- To verify a single piece of data, you only need O(log N) hashes (not the full dataset)
- Easy to find which pieces of data differ between two nodes

**Use cases in distributed systems:**
- **Cassandra anti-entropy repair**: Two replicas compare Merkle trees of their data to find diverged rows and sync only the differing parts
- **Git**: Uses Merkle trees internally (each commit is a hash of its tree + parent commit hashes)
- **Bitcoin/Blockchain**: Transaction hashes in a Merkle tree — the root hash goes in the block header
- **IPFS**: Content-addressable storage uses Merkle DAGs

---

### Schema Migration (Zero-Downtime)

**The problem:** You need to add a column, rename a field, or change a data type on a production DB with millions of rows — without downtime.

**The naive approach:** `ALTER TABLE users ADD COLUMN phone VARCHAR(20)` on a 100M row table → table lock → minutes of downtime. Never do this in production.

**The expand-contract (Blue-Green) pattern:**

*Step 1 — Expand:* Add new column/table as nullable. Both old and new code paths work.
```sql
ALTER TABLE users ADD COLUMN phone_v2 VARCHAR(20);  -- fast, non-blocking
```

*Step 2 — Migrate data:* Backfill in batches (don't lock the table):
```sql
UPDATE users SET phone_v2 = phone WHERE id BETWEEN 1 AND 10000;
-- repeat in chunks with a delay
```

*Step 3 — Deploy new code:* New code writes to `phone_v2`. Dual-write during transition.

*Step 4 — Contract:* Once all code uses `phone_v2`, drop `phone`.
```sql
ALTER TABLE users DROP COLUMN phone;
```

**Tools:**
- **Flyway / Liquibase** (Java ecosystem) — migration file management
- **golang-migrate** — popular for Go
- **Mongoose migrations** — Node.js/MongoDB
- **pt-online-schema-change** (Percona) — alters MySQL tables without locking

> [!WARNING]
> Never deploy schema changes and application code changes simultaneously. Always schema first, then code, then cleanup. This is the most common cause of production incidents during releases.

---

### Chaos Engineering

The practice of **intentionally injecting failures** into a system to verify it handles them gracefully.

**Process:**
1. Define "steady state" (normal behavior, metrics baseline)
2. Hypothesize: "System will maintain steady state even if X fails"
3. Inject failure (kill a pod, add latency to DB calls, drop network packets)
4. Observe: Does the system recover? Do error rates spike? Is the fallback triggered?
5. Fix weaknesses found

**Tools:**
- **Netflix Chaos Monkey**: Randomly kills production instances (famous)
- **Chaos Toolkit**: Open-source framework
- **AWS Fault Injection Simulator (FIS)**: Managed chaos for AWS

**What to test:**
- Single node failure → Does the load balancer reroute traffic?
- DB primary failure → Does failover to replica happen automatically?
- High latency on a downstream service → Does the circuit breaker trip?
- Cache unavailability → Does the system degrade gracefully (serve from DB)?

> [!NOTE]
> Netflix runs Chaos Monkey in production. Most companies run it in staging. In interviews, mentioning chaos engineering shows you think about resilience proactively.

---

### Data Consistency Patterns in NoSQL

How distributed NoSQL databases repair inconsistency:

**Read Repair:**
When a client reads from multiple replicas and gets different values, the coordinator detects the inconsistency and updates the stale replica with the latest value (in the background, after responding to the client).
- Used by: Cassandra, Dynamo

**Hinted Handoff:**
If a replica node is temporarily down during a write, the coordinator stores a "hint" (the write + destination node) locally. When the down node comes back, the coordinator replays the hint to bring it up to date.
- This is how Cassandra achieves high write availability even with node failures

**Anti-Entropy (Merkle Tree Comparison):**
Periodically, nodes compare their Merkle trees to detect and repair diverged data. Used for full reconciliation, not real-time repair.

**Quorum Reads/Writes:**
- Write to W out of N replicas → write is considered successful
- Read from R out of N replicas → take the latest value
- If `R + W > N` → you are guaranteed to read data from at least one node that has the latest write (strong consistency)
- Cassandra default: N=3, W=1, R=1 (fast but eventually consistent). For strong: W=2, R=2.

---

## 📚 Suggested Resources

| Resource | Why |
|----------|-----|
| ByteByteGo — "Design a Unique ID Generator" (YouTube) | Snowflake deep dive |
| ByteByteGo — "Bloom Filter" explainer | Best visual explanation |
| *DDIA* Ch. 5 (Replication — read repair, hinted handoff) | Authoritative source |
| LaunchDarkly blog — feature flags best practices | Practical feature flag patterns |
| Netflix Tech Blog — Chaos Engineering | Original chaos engineering blog |
| PlanetScale blog — zero-downtime schema migrations | MySQL-focused, practical |

---

## ✅ Progress Tracker

- [ ] Snowflake ID structure — explain 64-bit layout, properties, clock skew problem
- [ ] UUID v4 vs UUID v7 vs Snowflake — pick the right one for context
- [ ] Bloom filter — false positives vs false negatives, use cases
- [ ] Consistent hashing + virtual nodes — why vnodes improve distribution
- [ ] Gossip protocol — high-level: how Cassandra uses it for failure detection
- [ ] RPO vs RTO — can define and choose DR strategy given requirements
- [ ] Feature flags — 4 types, how canary release works with flags
- [ ] Back-pressure — what it is, strategies (drop/throttle/buffer/block)
- [ ] Merkle trees — use in Cassandra repair and Git
- [ ] Zero-downtime schema migration — expand-contract pattern
- [ ] Chaos engineering — what it tests and why it matters
- [ ] Quorum reads/writes — R + W > N for strong consistency

---

*Back to index → [[System Design Roadmap]]*
*Next → [[Phase 5 - High-Level Design Patterns]]*
