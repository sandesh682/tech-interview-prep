---
tags: [system-design, phase-0, prerequisites, mindset]
status: not-started
phase: 0
---

# Phase 0 — Prerequisites & Mindset

> [!IMPORTANT]
> Most candidates fail system design not because they don't know enough — but because they **don't know how to think out loud, ask the right questions, and structure an open-ended answer**. This phase fixes that before you learn a single component.

---

## 🧭 Overview

System design interviews are fundamentally different from coding interviews. There is **no single correct answer**. The interviewer is evaluating:
- How you handle ambiguity
- Whether your trade-offs are deliberate and justified
- How you communicate complex ideas simply
- Whether you think like a senior engineer, not just a code writer

This phase sets your **mental model** before you dive into components.

---

## 📋 Topics Table

| Topic | Level | Why It Matters | Est. Time |
|-------|-------|----------------|-----------|
| What interviewers actually evaluate | Must-Know | Changes your entire approach | 1 hr |
| How to structure a system design answer | Must-Know | Without structure, you'll ramble | 2 hrs |
| Back-of-envelope estimation | Must-Know | Shows engineering instincts | 2 hrs |
| Key numbers every engineer should know | Must-Know | Foundation for all capacity estimates | 1 hr |
| Trade-off thinking (no perfect system) | Must-Know | Senior SDEs must justify every choice | 1 hr |
| Reading engineering blogs | Good-to-Know | Real-world grounding | Ongoing |

---

## 📖 Detailed Notes

### What Interviewers Actually Evaluate

Interviewers at senior level look for **four things**:

1. **Clarification skills** — Do you ask the right questions before designing? (Requirements, scale, constraints)
2. **Structured thinking** — Can you move from vague to concrete in a logical sequence?
3. **Trade-off awareness** — Do you know *why* you're making a choice, not just *what* to choose?
4. **Communication** — Can a non-expert understand your design?

> [!WARNING]
> Jumping straight into the solution (e.g., "I'll use Redis and Kafka") without understanding requirements is one of the **top red flags** interviewers report.

---

### How to Structure a System Design Answer

Use the **RESHADED framework** (detailed in [[Phase 8 - Interview Strategy & Practice]]):

1. **R**equirements — Functional + Non-functional (scale, latency, availability)
2. **E**stimation — Users, QPS, storage, bandwidth (back-of-envelope)
3. **S**ystem interface — API design (what endpoints/events does this system expose?)
4. **H**igh-level design — Draw the big boxes first
5. **A**rchitecture deep-dive — Pick 2–3 critical components and go deep
6. **D**ata model — Schema, storage choice, indexing
7. **E**dge cases — Failures, bottlenecks, hot spots
8. **D**eep dive on non-functionals — Scaling, caching, monitoring

> [!TIP]
> Practice saying your thought process out loud. The words matter as much as the diagram.

---

### Back-of-Envelope Estimation

This is a non-negotiable senior-level skill. Interviewers use it to test whether your instincts are calibrated.

**The process:**
1. Estimate **daily active users (DAU)**
2. Convert to **QPS** (Queries Per Second): `DAU × requests_per_user_per_day / 86400`
3. Estimate **storage per day** = `QPS × avg_payload_size × 86400`
4. Estimate **bandwidth**: `QPS × payload_size`

**Example — Twitter-like system:**
- 100M DAU, 10 reads + 1 write per user per day
- Write QPS = `100M × 1 / 86400 ≈ ~1,200 QPS`
- Read QPS = `100M × 10 / 86400 ≈ ~12,000 QPS`
- If each tweet = 1 KB → `1,200 × 1KB = 1.2 MB/s write throughput`

> [!TIP] You likely know this — just revise
> As a MERN developer, you've done rough sizing before. The difference here is you need to be **fluent and fast** — aim to estimate in under 3 minutes.

---

### Key Numbers Every Engineer Should Know

Memorise these. They appear in almost every estimation:

| Metric                                     | Value    |     |
| ------------------------------------------ | -------- | --- |
| 1 day in seconds                           | ~86,400  |     |
| 1 month in seconds                         | ~2.5M    |     |
| 1 year in seconds                          | ~31.5M   |     |
| Latency: L1 cache                          | ~1 ns    |     |
| Latency: RAM read                          | ~100 ns  |     |
| Latency: SSD random read                   | ~100 µs  |     |
| Latency: HDD random read                   | ~10 ms   |     |
| Latency: Network round-trip (same DC)      | ~0.5 ms  |     |
| Latency: Network round-trip (cross-region) | ~150 ms  |     |
| Average HTTP request payload               | ~1–10 KB |     |
| Average image                              | ~300 KB  |     |
| Average video (1 min, compressed)          | ~10 MB   |     |

# 🧠 BOTE - Numbers & Storage Cheat Sheet

---

## 🔢 Number Scale

| Name      | Value              | Power of 10 | Shortcut |
|-----------|--------------------|-------------|----------|
| Thousand  | 1,000              | 10^3        | 1K       |
| Million   | 1,000,000          | 10^6        | 1M       |
| Billion   | 1,000,000,000      | 10^9        | 1B       |
| Trillion  | 1,000,000,000,000  | 10^12       | 1T       |

---

## 💾 Storage Units

| Unit | Full Form  | Bytes        | Power |
|------|-----------|-------------|-------|
| B    | Byte      | 1           | 10^0  |
| KB   | Kilobyte  | 1,000       | 10^3  |
| MB   | Megabyte  | 1,000,000   | 10^6  |
| GB   | Gigabyte  | 1,000,000,000 | 10^9  |
| TB   | Terabyte  | 10^12       | 10^12 |
| PB   | Petabyte  | 10^15       | 10^15 |
| EB   | Exabyte   | 10^18       | 10^18 |

---

## ⚡ Golden BOTE Shortcuts

- 1 KB  ≈ 10^3 bytes  
- 1 MB  ≈ 10^6 bytes  
- 1 GB  ≈ 10^9 bytes  

---

## 🚀 Must-Remember Patterns

- 1M × 1KB ≈ 1GB  
- 1B × 1KB ≈ 1TB  
- 1M × 1MB ≈ 1TB  

---

## 🔄 Conversion Rules

- Each step up   → ÷1000  
- Each step down → ×1000  

### Examples:
- 1000 MB = 1 GB  
- 5000 MB = 5 GB  
- 1 GB = 1000 MB  

---

## 🧩 Combined Thinking (Most Important)

| Users | Per User Data | Total |
|------|--------------|-------|
| 1M   | 1 KB         | 1 GB  |
| 10M  | 1 KB         | 10 GB |
| 1M   | 1 MB         | 1 TB  |
| 100M | 10 KB        | 1 TB  |

---

## 🎯 Mental Model (Interview Trick)

Million (10^6) + KB (10^3) = GB (10^9)

👉 Just add exponents

---

## ⚠️ Important Notes

- Use base 10 (not 1024) for interviews  
- Focus on approximation, not exact values  
- Speed > accuracy in BOTE  

---

> [!NOTE]
> These numbers are approximations. The goal isn't precision — it's **order of magnitude** accuracy.

---

### Trade-Off Thinking

In system design, a **tradeoff** is the idea that improving one quality of a system often comes at the cost of another. There's rarely a perfect solution — every architectural choice involves giving something up to gain something else.

| Trade-Off | Option A | Option B |
|-----------|----------|----------|
| Consistency vs Availability | Strong consistency (SQL, ZooKeeper) | High availability (NoSQL, eventual) |
| Latency vs Throughput | Low latency (synchronous, pre-computed) | High throughput (async, batched) |
| Simplicity vs Flexibility | Monolith | Microservices |
| Cost vs Performance | Commodity hardware + sharding | High-end hardware |
| Read vs Write optimization | Denormalized, cached reads | Normalized writes |

# ⚖️ System Design - Trade-offs Cheat Sheet

---

## 🧠 Core Principle

> Every system design decision = Trade-off

You are ALWAYS optimizing for something and sacrificing something else.

---

## ⚖️ 1. CAP Theorem

| Choice | Meaning | You Sacrifice |
|------|--------|-------------|
| CP   | Consistency + Partition Tolerance | Availability |
| AP   | Availability + Partition Tolerance | Consistency |
| CA   | Consistency + Availability | Not realistic in distributed systems |

### Quick Intuition:
- Banking → CP (correctness matters)
- Social Media → AP (availability matters)

---

## ⚡ 2. Consistency vs Availability

| Consistency | Availability |
|------------|-------------|
| Same data everywhere | System always responds |
| Slower | Faster |
| Safer | Better UX |

👉 Trade-off:
- Strong consistency → delays
- Eventual consistency → stale data

---

## 🚀 3. Latency vs Throughput

| Latency | Throughput |
|--------|-----------|
| Time per request | Requests per second |
| Low = fast response | High = handles more load |

👉 Trade-off:
- Batch processing → high throughput, high latency  
- Real-time → low latency, lower throughput  

---

## 💾 4. Storage vs Computation

| Storage Heavy | Compute Heavy |
|--------------|--------------|
| Precompute data | Compute on demand |
| Fast reads | Less storage |
| More space | More CPU |

👉 Example:
- Caching → more storage, less computation

---

## 🧩 5. Read vs Write Optimization

| Read Optimized | Write Optimized |
|---------------|----------------|
| Faster reads | Faster writes |
| Denormalized | Normalized |
| More duplication | Less duplication |

👉 Example:
- News Feed → Read heavy  
- Logging system → Write heavy  

---

## 🔁 6. Sync vs Async Processing

| Sync | Async |
|-----|------|
| Immediate response | Background processing |
| Slower | Faster response |
| Reliable | Eventually consistent |

👉 Example:
- Payment → Sync  
- Email sending → Async  

---

## 📦 7. SQL vs NoSQL

| SQL | NoSQL |
|----|------|
| Strong consistency | Flexible schema |
| Joins | No joins |
| Vertical scaling | Horizontal scaling |

👉 Trade-off:
- SQL → correctness  
- NoSQL → scalability  

---

## ⚡ 8. Caching Trade-offs

| Benefit | Cost |
|--------|------|
| Faster reads | Stale data |
| Reduced DB load | Cache invalidation complexity |

👉 Hardest problem:
> Cache invalidation 😄

---

## 🌍 9. Horizontal vs Vertical Scaling

| Vertical | Horizontal |
|---------|-----------|
| Bigger machine | More machines |
| Easy | Complex |
| Limited | Scalable |

---

## 🔐 10. Security vs Performance

| Secure | Fast |
|-------|------|
| Encryption | Less overhead |
| Authentication | Lower latency |

---

## 🎯 Interview Strategy

When asked "Why this design?"

👉 Always answer like this:

1. "We choose X because..."
2. "We sacrifice Y..."
3. "This is acceptable because..."

---

## 💡 Example Answer

"We use caching to reduce latency, sacrificing some consistency.  
This is acceptable because slight staleness is fine for this use case."

---

## 🧠 Golden Rule

> If you are NOT talking about trade-offs, you are NOT doing system design.


> [!TIP]
> When you make a choice, always say: *"I'm choosing X over Y because of [reason]. The trade-off I'm accepting is [downside]."* This is what makes you sound senior.

---

## 📚 Suggested Resources

| Resource | Why |
|----------|-----|
| ByteByteGo — "System Design Interview Tips" (YouTube) | Best intro to interview structure |
| Gaurav Sen — "Introduction to System Design" playlist | Intuition-first explanations |
| *System Design Interview Vol 1* — Alex Xu (Ch. 1–2) | Estimation + framework |
| Latency Numbers Everyone Should Know — Colin Scott (GitHub) | Quick numbers reference |

---

## ✅ Progress Tracker

- [x] Understand what interviewers evaluate at Senior SDE level
- [x] Learn and internalize the RESHADED (or similar) framework
- [x] Practice 3 back-of-envelope estimations from scratch
- [ ] Memorise the key latency + scale numbers
- [ ] Articulate 5 trade-offs with justification (write them down)
- [ ] Read one engineering blog post end-to-end (Uber, Netflix, etc.)

---

*Next → [[Phase 1 - Foundations]]*
