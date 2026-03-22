---
tags: [system-design, phase-4, distributed-systems, cap-theorem, consistency, consensus]
status: not-started
phase: 4
---

# Phase 4 — Distributed Systems Concepts

> [!IMPORTANT]
> This phase separates mid-level from senior-level candidates. You don't need to implement Raft consensus — but you need to **speak intelligently about failure modes, consistency trade-offs, and why distributed systems are fundamentally hard**.

---

## 🧭 Overview

Once you scale beyond a single server, you enter distributed systems territory: multiple nodes, networks that can fail, clocks that drift, and processes that can crash at any moment. This phase covers the theoretical foundations that explain *why* systems behave the way they do and *how* to design around the inherent challenges.

---

## 📋 Topics Table

| Topic | Level | Why It Matters | Est. Time |
|-------|-------|----------------|-----------|
| CAP Theorem | Must-Know | Foundation of all distributed DB trade-offs | 1.5 hrs |
| Consistency models (strong, eventual, causal) | Must-Know | Know what your system promises clients | 2 hrs |
| PACELC Theorem | Good-to-Know | More nuanced than CAP | 1 hr |
| Fallacies of distributed computing | Must-Know | Understand what can go wrong | 1 hr |
| Fault tolerance — replication, redundancy | Must-Know | How systems stay up despite failures | 1.5 hrs |
| Consensus algorithms (Paxos, Raft) | Good-to-Know | Leader election, ZooKeeper, etcd | 1.5 hrs |
| Clock synchronization & ordering | Must-Know | Event ordering without global clock | 1.5 hrs |
| Idempotency & exactly-once semantics | Must-Know | Safe retries in distributed systems | 1.5 hrs |
| Circuit Breaker pattern | Must-Know | Preventing cascading failures | 1 hr |
| Saga pattern (for distributed transactions) | Must-Know | Transactions across microservices | 2 hrs |
| Two-Phase Commit (2PC) | Must-Know | Distributed transaction protocol | 1 hr |

---

## 📖 Detailed Notes

### CAP Theorem

A distributed system can only guarantee **2 of 3** properties simultaneously:

- **C**onsistency — Every read sees the most recent write (or an error)
- **A**vailability — Every request receives a response (not necessarily the latest data)
- **P**artition Tolerance — System continues operating despite network partitions (lost messages between nodes)

> [!IMPORTANT]
> Network partitions **will happen**. So in practice, you're always choosing between **CP** and **AP** — Partition Tolerance is mandatory.

| Choice | Behavior During Partition | Examples |
|--------|--------------------------|----------|
| **CP** | Refuse requests to avoid returning stale data | ZooKeeper, HBase, MongoDB (with `writeConcern: majority`) |
| **AP** | Return potentially stale data to stay available | DynamoDB (default), Cassandra, CouchDB |

**How to use in interviews:**
> "For this payment service, I'll choose a CP database — I'd rather reject a request than process a payment twice or miss one."
> "For the social feed, AP is fine — showing a post 500ms late is acceptable."

> [!NOTE]
> CAP Theorem is often misunderstood. "Consistency" here means **linearizability**, not eventual consistency. Don't confuse it with ACID consistency.

---

### Consistency Models

From strongest to weakest:

1. **Linearizability (Strong Consistency)** — Reads always see the latest write. Feels like a single machine. Expensive. (Example: reading from primary only in PostgreSQL)

2. **Sequential Consistency** — Operations appear to execute in some sequential order, but that order may not match real-time. Less strict than linearizability.

3. **Causal Consistency** — Causally related operations appear in order. If A caused B, everyone sees A before B. (Example: "Read your own writes")

4. **Eventual Consistency** — Given no new updates, all replicas will converge to the same value. No guarantee on *when*. (Example: DNS, DynamoDB default, MongoDB secondary reads)

**"Read your own writes" consistency:** After a user writes, subsequent reads from the same user see that write. Implemented by routing that user's reads to primary for a brief window.

---

### PACELC Theorem

An extension of CAP: Even when there is **no partition**, you still trade off between **Latency (L)** and **Consistency (C)**.

- `PAC`: Partition → Choose between Availability and Consistency
- `ELC`: Else (no partition) → Choose between Latency and Consistency

| System | Partition choice | Normal operation choice |
|--------|-----------------|------------------------|
| DynamoDB | AP | EL (low latency, eventual) |
| Cassandra | AP | EL |
| MongoDB | CP | EC (configurable) |
| PostgreSQL | CP | EC |

---

### Fallacies of Distributed Computing

Every distributed system designer must internalize these (Peter Deutsch, Sun Microsystems):

1. The network is reliable
2. Latency is zero
3. Bandwidth is infinite
4. The network is secure
5. Topology doesn't change
6. There is one administrator
7. Transport cost is zero
8. The network is homogeneous

> [!WARNING]
> These fallacies explain why naïve distributed system designs fail. Defensive design: assume any network call can fail, time out, or succeed silently while the effect was not applied.

---

### Fault Tolerance

**Types of failures:**
- **Fail-stop**: Node stops and doesn't send incorrect data (crash failure)
- **Byzantine failure**: Node sends incorrect data (hardware corruption, malicious actors) — hardest to handle
- **Network partition**: Nodes can't communicate (neither fails, just disconnected)

**Fault tolerance techniques:**
- **Replication** — multiple copies of data/service
- **Redundancy** — hot standbys, N+1 capacity
- **Timeouts** — don't wait forever; fail fast
- **Retries with exponential backoff** — retry failed calls but back off to avoid overwhelming a recovering service
- **Bulkhead pattern** — isolate failures; if one service pool is exhausted, others still work
- **Health checks + auto-restart** — Kubernetes does this for you

**SLA / SLO / SLI:**
- **SLI** (Service Level Indicator): Actual measurement (e.g., 99.5% uptime last month)
- **SLO** (Service Level Objective): Target (e.g., 99.9% uptime)
- **SLA** (Service Level Agreement): Contractual promise (e.g., 99.9% or we give refunds)

**Nines of availability:**

| Uptime | Downtime/year |
|--------|--------------|
| 99% ("2 nines") | 3.65 days |
| 99.9% ("3 nines") | 8.7 hours |
| 99.99% ("4 nines") | 52 minutes |
| 99.999% ("5 nines") | 5.26 minutes |

---

### Consensus Algorithms

Consensus = getting multiple nodes to agree on a single value, even in the presence of failures.

**Why it matters:** Leader election, distributed locking, atomic broadcast, config management.

**Paxos:** The classic consensus algorithm. Correct but notoriously complex to implement and understand.

**Raft:** Designed to be more understandable than Paxos. Used in etcd (Kubernetes' config store) and CockroachDB.
- **Leader election**: Nodes vote; the node with the most votes becomes leader
- **Log replication**: Leader appends entries to its log, replicates to followers
- **Commit**: Entry is committed once a majority of nodes acknowledge it

**ZooKeeper:** Uses ZAB (ZooKeeper Atomic Broadcast), similar to Raft. Used for: Kafka broker coordination, leader election, distributed locks.

> [!NOTE]
> You don't need to implement Raft. But know: "I'd use etcd/ZooKeeper for leader election because it provides consensus guarantees — a distributed system with two leaders is worse than one with no leader."

---

### Clock Synchronization & Event Ordering

Distributed systems don't have a global clock. This creates ordering problems: if two events happen at nearly the same time on different nodes, which came first?

**Physical clocks drift** — NTP keeps them close but not identical (~millisecond drift). Google Spanner uses atomic clocks + GPS ("TrueTime") to bound clock uncertainty.

**Logical Clocks:**
- **Lamport Timestamps** — each event has a counter. Counter increments on every event. On message receipt, `max(local, received) + 1`. Provides causal ordering.
- **Vector Clocks** — array of counters, one per node. Tracks causality precisely. More expensive but accurate.

**Happens-before relationship:** Event A "happens before" B if: A and B are on the same node and A occurs first, OR A sends a message that B receives.

---

### Idempotency & Exactly-Once Semantics

**Idempotent operation:** Can be safely executed multiple times with the same result. `PUT /users/123 {name: "Sandesh"}` is idempotent. `POST /payments` is not (two calls = two charges).

**Why it matters:** Networks are unreliable. Clients retry. You must design for retries.

**Idempotency key pattern:**
1. Client generates a unique `idempotency_key` (UUID) per operation
2. Server checks if this key has been processed before
3. If yes → return cached result. If no → process and store the result with the key

> [!TIP] You likely know this — just revise
> You've dealt with duplicate events in Node.js APIs. In interviews, proactively mention: "I'd add an idempotency key to payment and order creation endpoints to handle client retries safely."

---

### Circuit Breaker Pattern

Prevents cascading failures: if a downstream service is unhealthy, stop calling it — fail fast instead of queuing up requests that will timeout.

**States:**
- **Closed** (normal): Requests pass through. Failure count tracked.
- **Open** (tripped): After threshold failures, all requests fail immediately without calling the downstream service.
- **Half-Open** (testing): After a timeout, allow some requests through. If they succeed → Close. If they fail → Open again.

**Libraries:** Hystrix (Netflix, now deprecated), Resilience4j (Java), `opossum` (Node.js)

---

### Saga Pattern (Distributed Transactions)

A distributed transaction spans multiple services. 2PC is one option (see below) but it's blocking. Sagas are the modern alternative.

A **Saga** is a sequence of local transactions. If one fails, compensating transactions undo the previous steps.

**Choreography-based Saga:** Each service publishes events; the next service listens and proceeds. No central coordinator. Simple, but hard to track.

**Orchestration-based Saga:** A central **Saga Orchestrator** tells each service what to do and handles failures. More control, easier to visualize, single point of failure.

**Example — Order placement:**
1. Order Service: Create order (local transaction)
2. Payment Service: Charge card → if fail → cancel order
3. Inventory Service: Reserve stock → if fail → refund payment + cancel order
4. Notification Service: Send confirmation

> [!NOTE]
> Sagas provide **eventual consistency**, not ACID. Make sure interviewers know you understand this trade-off.

---

### Two-Phase Commit (2PC)

A protocol for distributed transactions that guarantees atomicity across multiple nodes.

**Phase 1 (Prepare):** Coordinator asks all participants: "Can you commit?" Each participant locks resources and votes yes/no.
**Phase 2 (Commit/Rollback):** If all vote yes → Coordinator sends Commit. If any vote no → Coordinator sends Rollback.

**Problems with 2PC:**
- **Blocking protocol**: If coordinator crashes after Phase 1, participants are locked indefinitely
- **Single point of failure**: The coordinator
- **Not partition-tolerant**: A network partition during Phase 2 leaves participants in limbo

> [!TIP]
> 2PC is rarely used in modern distributed systems (beyond XA transactions in legacy systems). Sagas + idempotency are the modern answer. Know 2PC to explain *why* we moved away from it.

---

## 📚 Suggested Resources

| Resource | Why |
|----------|-----|
| *DDIA* Ch. 7 (Transactions) + Ch. 8 (Trouble with Distributed Systems) + Ch. 9 (Consistency) | The canonical reference |
| Gaurav Sen — "CAP Theorem" (YouTube) | Best intuitive explanation |
| Martin Kleppmann — "Distributed Systems" lecture series (YouTube, Cambridge) | Free, university-level depth |
| ByteByteGo — "Saga Pattern" explainer | Visual walkthrough |
| The Raft consensus paper — raft.github.io | Read the summary, not the full paper |

---

## ✅ Progress Tracker

- [ ] CAP Theorem — explain with real DB examples (CP vs AP choice)
- [ ] Consistency models spectrum (strong → eventual)
- [ ] PACELC — explain the latency-consistency trade-off
- [ ] 8 Fallacies of distributed computing
- [ ] Failure types: fail-stop, Byzantine, network partition
- [ ] Retries with exponential backoff + jitter
- [ ] Nines of availability — know the downtime numbers
- [ ] Raft/Paxos — high-level understanding, when to use ZooKeeper/etcd
- [ ] Lamport timestamps and vector clocks concept
- [ ] Idempotency key pattern — can implement in an API
- [ ] Circuit Breaker — 3 states and when to trip
- [ ] Saga pattern — choreography vs orchestration
- [ ] 2PC — understand why it's blocking and why Sagas replaced it

---

*Next → [[Phase 4.5 - Critical Components Often Missed]]* (covers Snowflake IDs, Bloom Filters, Feature Flags, DR/RTO/RPO and other frequently missed topics)
