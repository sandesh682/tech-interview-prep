---
tags: [system-design, roadmap, index, senior-sde]
status: in-progress
type: index
last-updated: 2026-03-20
---

# 🗺️ System Design Interview Roadmap
### Senior SDE — MERN + AWS + Micro-Frontend Background

> [!IMPORTANT]
> This vault is tailored for a **Senior SDE (6+ years)** targeting both **service-based** (TCS, Infosys, Wipro, Capgemini) and **product-based** (Swiggy, Zepto, PhonePe, Atlassian, Freshworks, Razorpay) companies in India. Adjust depth based on your target company tier.

---

## 📋 Phase Index

| # | Phase | Focus | Est. Time | Status |
|---|-------|-------|-----------|--------|
| 0 | [[Phase 0 - Prerequisites & Mindset]] | Mental models, how to think in systems | 3 days | ⬜ |
| 1 | [[Phase 1 - Foundations]] | Networking, HTTP, DNS, CDN, Load Balancers | 1 week | ⬜ |
| 2 | [[Phase 2 - Core Building Blocks]] | Caches, Queues, Databases, APIs | 1.5 weeks | ⬜ |
| 2.5 | [[Phase 2.5 - LLD & SOLID Principles]] | SOLID, Design Patterns, Class Design, LLD Cases | 1.5 weeks | ⬜ |
| 3 | [[Phase 3 - Data & Storage Deep Dive]] | SQL vs NoSQL, sharding, replication | 1 week | ⬜ |
| 4 | [[Phase 4 - Distributed Systems Concepts]] | CAP, consistency, consensus, failures | 1 week | ⬜ |
| 4.5 | [[Phase 4.5 - Critical Components Often Missed]] | Snowflake IDs, Bloom Filters, Feature Flags, DR/RPO/RTO | 1 week | ⬜ |
| 5 | [[Phase 5 - High-Level Design Patterns]] | Microservices, event-driven, CQRS, Sagas | 1 week | ⬜ |
| 6 | [[Phase 6 - Real-World System Design Cases]] | 10 full design walkthroughs | 2 weeks | ⬜ |
| 7 | [[Phase 7 - Advanced & Specialized Topics]] | ML systems, real-time, search, payments | 1 week | ⬜ |
| 8 | [[Phase 8 - Interview Strategy & Practice]] | Frameworks, mock practice, communication | 4 days | ⬜ |

**Total Estimated Time: 11–13 weeks** (1–2 hrs/day)

---

## ⏱️ Study Timeline

### 🔥 Aggressive Track — 8 Weeks (for urgent deadlines)

```
Week 1  → Phase 0 + Phase 1 (Mindset + Foundations)
Week 2  → Phase 2 (Core Building Blocks) + Phase 2.5 LLD highlights
Week 3  → Phase 2.5 LLD deep dive (SOLID + Design Patterns + 3 LLD cases)
Week 4  → Phase 3 + Phase 4 (Data + Distributed)
Week 5  → Phase 4.5 (Missing topics) + Phase 5 (Design Patterns)
Week 6  → Phase 6 (Real-World Cases — do at least 6)
Week 7  → Phase 7 highlights
Week 8  → Phase 8 (Strategy + Mock interviews)
```

### 🧘 Balanced Track — 13 Weeks (recommended)

```
Week 1        → Phase 0 + Phase 1
Week 2–3      → Phase 2 (Core Building Blocks)
Week 4–5      → Phase 2.5 (LLD & SOLID — dedicate real time here)
Week 6        → Phase 3 (Data & Storage)
Week 7        → Phase 4 (Distributed Systems)
Week 8        → Phase 4.5 (Critical Missing Topics)
Week 9        → Phase 5 (High-Level Design Patterns)
Week 10–11    → Phase 6 (all 10 real-world cases)
Week 12       → Phase 7 (Advanced Topics)
Week 13       → Phase 8 + Final Mock Sessions
```

> [!TIP]
> Given your MERN + AWS background, **Phase 1 and parts of Phase 2** will be faster for you. Bank that saved time for Phase 4 and Phase 6.

---

## 📊 Overall Progress Tracker

### Phase Completion
- [ ] Phase 0 — Prerequisites & Mindset
- [ ] Phase 1 — Foundations
- [ ] Phase 2 — Core Building Blocks
- [ ] Phase 2.5 — LLD & SOLID Principles *(NEW)*
- [ ] Phase 3 — Data & Storage Deep Dive
- [ ] Phase 4 — Distributed Systems Concepts
- [ ] Phase 4.5 — Critical Components Often Missed *(NEW)*
- [ ] Phase 5 — High-Level Design Patterns
- [ ] Phase 6 — Real-World System Design Cases
- [ ] Phase 7 — Advanced & Specialized Topics
- [ ] Phase 8 — Interview Strategy & Practice

### Mock Interview Tracker
- [ ] Mock 1 — Design a URL Shortener
- [ ] Mock 2 — Design Twitter Feed
- [ ] Mock 3 — Design a Notification System
- [ ] Mock 4 — Design WhatsApp
- [ ] Mock 5 — Design Uber
- [ ] Mock 6 — Design a Payment Gateway
- [ ] Mock 7 — Design a Search Autocomplete
- [ ] Mock 8 — Design Netflix
- [ ] Mock 9 — Open-ended (pick a random case)
- [ ] Mock 10 — Full timed simulation (45 min)

### Key Milestones
- [ ] Can explain SOLID principles with a code example for each
- [ ] Can design a Parking Lot or Elevator system class diagram from scratch
- [ ] Can name and apply 5 GoF design patterns with real use cases
- [ ] Can explain CAP theorem with a real-world example in < 2 minutes
- [ ] Can draw a high-level design for any FAANG-style problem in 5 minutes
- [ ] Can explain Snowflake ID structure (64-bit layout) without notes
- [ ] Have practiced at least 2 mock interviews with a peer
- [ ] Revised all [[Phase 2 - Core Building Blocks]] topics at least twice
- [ ] Read at least 3 chapters of DDIA (Chapters 1, 5, 7 are the most critical)

---

## 🔗 Quick Reference Links

| Topic | File |
|-------|------|
| SOLID Principles | [[Phase 2.5 - LLD & SOLID Principles#SOLID Principles]] |
| Design Patterns (GoF) | [[Phase 2.5 - LLD & SOLID Principles#Design Patterns]] |
| LLD Cases | [[Phase 2.5 - LLD & SOLID Principles#LLD Case Parking Lot]] |
| Caching strategies | [[Phase 2 - Core Building Blocks#Caching]] |
| SQL vs NoSQL | [[Phase 3 - Data & Storage Deep Dive]] |
| CAP Theorem | [[Phase 4 - Distributed Systems Concepts]] |
| Snowflake ID Generation | [[Phase 4.5 - Critical Components Often Missed#Unique ID Generation]] |
| Bloom Filters | [[Phase 4.5 - Critical Components Often Missed#Bloom Filters]] |
| Feature Flags | [[Phase 4.5 - Critical Components Often Missed#Feature Flags]] |
| Disaster Recovery (RTO/RPO) | [[Phase 4.5 - Critical Components Often Missed#Disaster Recovery]] |
| Microservices patterns | [[Phase 5 - High-Level Design Patterns]] |
| Design Framework (RESHADED) | [[Phase 8 - Interview Strategy & Practice]] |

---

## 📚 Master Resource List

| Resource | Type | Priority |
|----------|------|----------|
| *Designing Data-Intensive Applications* (Kleppmann) | Book | ⭐⭐⭐ Must |
| ByteByteGo (Alex Xu) — YouTube + Newsletter | Video + Articles | ⭐⭐⭐ Must |
| Gaurav Sen — YouTube | Video | ⭐⭐⭐ Must |
| *System Design Interview Vol 1 & 2* (Alex Xu) | Book | ⭐⭐ High |
| High Scalability Blog — highscalability.com | Articles | ⭐⭐ High |
| Engineering blogs (Uber, Airbnb, Netflix, Swiggy) | Articles | ⭐⭐ High |
| Grokking System Design — educative.io | Course | ⭐ Optional |

---

> [!NOTE]
> This vault is a **living document**. Update the `status` frontmatter in each phase file as you progress: `not-started → in-progress → done`.

---

*Start here → [[Phase 0 - Prerequisites & Mindset]]*
