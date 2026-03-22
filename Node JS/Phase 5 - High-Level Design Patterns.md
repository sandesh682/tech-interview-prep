---
tags: [system-design, phase-5, microservices, event-driven, cqrs, design-patterns]
status: not-started
phase: 5
---

# Phase 5 — High-Level Design Patterns

> [!NOTE]
> Given your micro-frontend and AWS experience, you'll recognize many of these patterns. Focus on being able to **articulate trade-offs and failure modes**, not just describe what they are.

---

## 🧭 Overview

Architecture patterns are reusable solutions to recurring design problems. This phase covers the macro-level structural patterns that determine how your system's components are organized and how they communicate. These are the "shapes" you'll draw in your system design diagrams.

---

## 📋 Topics Table

| Topic | Level | Why It Matters | Est. Time |
|-------|-------|----------------|-----------|
| Monolith vs Microservices | Must-Know | Trade-off every design starts with | 1.5 hrs |
| Event-Driven Architecture | Must-Know | Decoupling, async systems | 2 hrs |
| CQRS (Command Query Responsibility Segregation) | Must-Know | Separating reads and writes | 1.5 hrs |
| Event Sourcing | Good-to-Know | Audit log, event replay | 1.5 hrs |
| API Gateway pattern | Must-Know | Microservices entry point | 1 hr |
| BFF (Backend for Frontend) | Must-Know | Mobile vs web different needs | 1 hr |
| Strangler Fig pattern | Good-to-Know | Migrating monolith to microservices | 1 hr |
| Sidecar / Service Mesh | Good-to-Know | Observability, traffic management | 1.5 hrs |
| Bulkhead pattern | Must-Know | Failure isolation | 1 hr |
| Outbox pattern | Must-Know | Reliable event publishing | 1.5 hrs |
| Micro-frontend architecture | Must-Know | You know this — articulate it precisely | 1 hr |

---

## 📖 Detailed Notes

### Monolith vs Microservices

**Monolith:**
- Single deployable unit
- Simple to develop, test, deploy initially
- Scales as a whole unit (vertical scaling or replicate entire app)
- As codebase grows: deployment bottlenecks, long build times, team coupling

**Microservices:**
- System split into independent services, each owning its data and deployable independently
- Pros: Independent scaling, independent deployment, technology flexibility per service, team autonomy
- Cons: Network overhead (latency, failures), operational complexity (service discovery, distributed tracing), distributed transaction complexity

**The decision framework for interviews:**
- Early-stage / small team → Monolith first (it's faster)
- High scale + multiple teams + different scaling needs → Microservices
- Rule: "We'll design with microservices because the problem statement implies multiple teams and independent scaling requirements for different parts of the system."

> [!TIP] You likely know this — just revise
> With your micro-frontend background, you understand the frontend equivalent. Apply the same "independent deployment + team autonomy" reasoning to backend services. Interviewers love when you draw the parallel.

**Decomposition strategies:**
- **By business capability**: User Service, Order Service, Payment Service
- **By subdomain (DDD)**: Domain-Driven Design bounded contexts
- **By volatility**: Extract services that change often or have different scaling needs first

---

### Event-Driven Architecture (EDA)

Services communicate via events rather than direct calls (synchronous REST/RPC).

**Components:**
- **Event Producers** — emit events when something happens (`order.placed`, `payment.processed`)
- **Event Broker** — Kafka, SQS, RabbitMQ
- **Event Consumers** — subscribe to events they care about

**Benefits:**
- Loose coupling — producer doesn't know who's consuming
- Temporal decoupling — consumer can be down; events wait in queue
- Easy to add new consumers without changing producer (open/closed principle)
- Natural audit log (events are a record of what happened)

**Drawbacks:**
- Eventual consistency — harder to reason about system state
- Debugging is harder (no call stack across async boundaries)
- Event schema evolution is tricky — consumers may be on different versions

**Event vs Command:**
- **Event**: "order.placed" (something happened, past tense, no expectation of response)
- **Command**: "place_order" (requesting an action, expecting a result)

> [!NOTE]
> Kafka is the default for event-driven architecture at scale. Know that each Kafka **topic** can have multiple **partitions**, and a **consumer group** processes each partition with exactly one consumer — enabling parallel processing.

---

### CQRS — Command Query Responsibility Segregation

Separate the **write model** (Commands) from the **read model** (Queries).

**Traditional model:** One database, one model, reads and writes use the same schema.

**CQRS:**
- **Command side**: Handles writes. Optimized for consistency. Often normalized.
- **Query side**: Handles reads. Optimized for query patterns. Often denormalized or pre-computed.
- Synced via **events** (the write side publishes events; the read side updates its projection)

**When to use:**
- Read and write workloads have very different scaling needs
- Complex read queries that can be pre-computed
- Systems that need an audit trail (pair with Event Sourcing)

**Example:** An e-commerce product page:
- Write side: `products` table in PostgreSQL (normalized, ACID)
- Read side: Pre-computed JSON blobs in Redis or Elasticsearch (denormalized, fast reads)
- Sync: Product updated → event → consumer updates Redis/ES

> [!WARNING]
> CQRS adds significant complexity. Don't reach for it unless there's a clear need. In interviews, propose it only when discussing high read/write asymmetry or complex query requirements.

---

### Event Sourcing

Instead of storing the current state, store the **sequence of events** that led to the current state.

**Example:** Instead of `{ balance: 1000 }`, store:
```
deposit(500)
withdraw(200)
deposit(700)
→ balance = 1000
```

**Benefits:**
- Complete audit history
- Replay events to reconstruct state at any point in time
- Natural fit with CQRS
- Can derive new projections from historical events

**Drawbacks:**
- Querying current state requires replaying events (mitigated with snapshots)
- Event schema evolution is complex
- Not suitable for all domains

---

### API Gateway Pattern

> [!TIP] You likely know this — just revise
> Already covered in [[Phase 2 - Core Building Blocks#API Gateway]]. Key addition for this phase: how it fits in the microservices picture.

In microservices, the API Gateway is the **single entry point** for all clients. It handles:
- Routing requests to the right service
- Authentication (so individual services don't need auth logic)
- Rate limiting
- Request aggregation (one API call → multiple service calls → aggregated response)
- Protocol translation (REST externally → gRPC internally)

---

### BFF — Backend for Frontend

A variation of API Gateway: instead of one gateway for all clients, have **one gateway per client type**.

**Why:** Mobile app needs different data than web app. Mobile needs less data (bandwidth constraints). Web dashboard needs richer data. A single API can't optimize for both.

**Structure:**
- `mobile-bff` — lightweight, compressed payloads, batched requests for mobile
- `web-bff` — richer responses, pre-aggregated data for dashboards
- `partner-api` — versioned, stable API for external partners

> [!TIP] You likely know this — just revise
> This is conceptually similar to your micro-frontend work, where each app consumes different data. BFF is the backend equivalent of a micro-frontend shell.

---

### Strangler Fig Pattern

A strategy for migrating a monolith to microservices incrementally, without a risky big-bang rewrite.

**How it works:**
1. New features are built as microservices (outside the monolith)
2. An API Gateway/proxy sits in front of both
3. Gradually, existing functionality is extracted from the monolith into services
4. The monolith "strangles" as more and more of it is replaced
5. Eventually, the monolith is gone (or just the parts that don't justify extraction remain)

**Benefit:** Zero-downtime migration. Always have a working system.

---

### Sidecar Pattern & Service Mesh

**Sidecar:** Each service instance has a co-located proxy container (sidecar) that handles cross-cutting concerns:
- Observability (metrics, tracing)
- Traffic management (retry, circuit breaking, timeout)
- Security (mTLS between services)

**Service Mesh:** A network of sidecar proxies (e.g., Envoy) coordinated by a control plane (e.g., Istio, Linkerd).
- Services communicate through their sidecars
- Policies (rate limit, retry) configured centrally in the control plane

> [!NOTE]
> In Kubernetes environments, service meshes are common at large organizations. For interviews at product companies (Swiggy, Razorpay), mentioning Istio shows operational maturity. Don't over-engineer — note it as an option for observability/security requirements.

---

### Bulkhead Pattern

Isolates components so that failure in one doesn't cascade to others. Named after ship compartments.

**Application-level bulkheads:**
- Separate thread pools / connection pools per downstream service
- If Service A's thread pool is exhausted (Service B is slow), Service C is unaffected

**Kubernetes-level bulkheads:**
- Separate namespaces, node pools, or clusters per criticality tier
- Ensures a runaway workload doesn't starve critical services

---

### Outbox Pattern

**Problem:** You want to write to a database AND publish an event atomically. If your app crashes between the two, you get inconsistency.

**Solution (Transactional Outbox):**
1. Write your business data AND an event record to the **same DB transaction** (in an `outbox` table)
2. A separate **Outbox Processor** reads from the outbox table and publishes events to Kafka/SQS
3. On success, mark events as published (or delete them)

**Result:** The event is published *if and only if* the DB transaction committed. No lost events, no phantom events.

This is critical for microservices that need reliable event publishing without distributed transactions.

---

### Micro-Frontend Architecture

> [!TIP] You likely know this — just revise
> Given your background, articulate this precisely for interviews.

**Core idea:** Frontend is broken into independently deployable units (micro-frontends), each owned by a separate team.

**Integration approaches:**
- **Module Federation (Webpack 5)** — load remote JS bundles at runtime; your most relevant experience
- **iframe isolation** — strong isolation but poor UX
- **Web Components** — standards-based, framework-agnostic
- **Server-side composition** — Edge Side Includes or server renders micro-frontends into a shell

**Trade-offs to discuss:**
- Bundle duplication (React loaded multiple times) → mitigated by shared libs in Module Federation
- Consistent UX across apps → shared design system / component library
- Independent deployment → CI/CD per micro-frontend, versioning strategy
- Cross-app communication → custom events, shared state store, or URL params

---

## 📚 Suggested Resources

| Resource | Why |
|----------|-----|
| Martin Fowler — martinfowler.com (CQRS, Event Sourcing, Strangler Fig articles) | Authoritative pattern descriptions |
| ByteByteGo — "Microservices vs Monolith" (YouTube) | Clear visual trade-off |
| Gaurav Sen — "Event-Driven Architecture" | Intuition-first explanation |
| Sam Newman — *Building Microservices* (book) | Comprehensive reference |
| *DDIA* Ch. 11 (Stream Processing) | Event streams at scale |

---

## ✅ Progress Tracker

- [ ] Monolith vs Microservices — justify choice with team/scale reasoning
- [ ] Event-Driven Architecture — producers, consumers, brokers, trade-offs
- [ ] Event vs Command distinction
- [ ] CQRS — when to apply, how read/write models sync
- [ ] Event Sourcing — benefits and when NOT to use it
- [ ] API Gateway responsibilities in microservices context
- [ ] BFF pattern — mobile vs web optimization
- [ ] Strangler Fig — migration strategy without big-bang rewrite
- [ ] Sidecar / Service Mesh — Istio/Envoy high-level understanding
- [ ] Bulkhead — thread pool isolation
- [ ] Outbox pattern — reliable event publishing with DB transactions
- [ ] Micro-frontend patterns — can discuss Module Federation trade-offs

---

*Next → [[Phase 6 - Real-World System Design Cases]]*
