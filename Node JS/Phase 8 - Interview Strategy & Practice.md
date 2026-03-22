---
tags: [system-design, phase-8, interview-strategy, mock, framework, communication]
status: not-started
phase: 8
---

# Phase 8 — Interview Strategy & Practice

> [!IMPORTANT]
> The best technical knowledge means nothing if you can't communicate it under pressure in 45 minutes. This phase is about performance, not learning. Practice until the framework is automatic.

---

## 🧭 Overview

This phase focuses on the **interview execution layer**: how to structure your 45-minute session, what to say when you're stuck, how to handle follow-up questions, and how to practice effectively. You also get a calibrated view of what different company tiers expect in India's market.

---

## 📋 Topics Table

| Topic | Level | Why It Matters | Est. Time |
|-------|-------|----------------|-----------|
| RESHADED framework (interview structure) | Must-Know | Your script for every interview | 2 hrs |
| Clarifying questions — the right ones | Must-Know | Shows senior thinking from minute 1 | 1 hr |
| Drawing diagrams effectively | Must-Know | Communication through visuals | 1 hr |
| Handling "go deeper" follow-ups | Must-Know | Interviewers probe your depth | 1 hr |
| What service-based companies expect | Must-Know | Calibrate depth for TCS, Infosys, etc. | 30 min |
| What product-based companies expect | Must-Know | Calibrate for Swiggy, Razorpay, etc. | 30 min |
| Mock interview practice guide | Must-Know | Practice > reading | Ongoing |
| Common mistakes and how to avoid them | Must-Know | Don't fail for avoidable reasons | 1 hr |
| Tailoring your MERN/AWS background | Must-Know | Use your experience strategically | 1 hr |

---

## 📖 Detailed Notes

### The RESHADED Framework

Use this as your mental script for every system design interview. Spend approximately the time noted for a 45-minute interview.

```
R — Requirements      (5 min)
E — Estimation        (5 min)
S — System Interface  (3 min)
H — High-Level Design (10 min)
A — Architecture      (10 min)
D — Data Model        (5 min)
E — Edge Cases        (5 min)
D — Deep Dive         (7 min)
```

---

#### R — Requirements (5 min)

Never skip this. Ask until you have:

**Functional requirements (what the system does):**
- "Should users be able to edit or delete a post?"
- "Do we need to support direct messages or only group chats?"
- "Should the URL shortener support custom aliases?"

**Non-functional requirements (how well it does it):**
- Scale: "How many daily active users are we designing for?"
- Latency: "What's the acceptable latency for reading the feed? < 200ms?"
- Availability: "Do we need 99.9% or 99.99% uptime?"
- Consistency: "Is it okay if users see a slightly stale feed?"
- Geographic: "Is this a global system or India-only for now?"

> [!TIP]
> Start with: *"Before I start designing, let me clarify the requirements."* Interviewers love this — it immediately signals seniority.

---

#### E — Estimation (5 min)

Do rough back-of-envelope math. Key questions:
- DAU → QPS (read and write separately)
- Storage per day/year
- Bandwidth

State your assumptions out loud: *"I'll assume 50M DAU and a 10:1 read-to-write ratio."*

Don't get bogged down in precision. Move on once you have the order of magnitude.

---

#### S — System Interface (3 min)

Define the key API endpoints:
```
POST /tweets          → create a tweet
GET  /feed/{userId}   → get user's feed (paginated)
POST /follow          → follow a user
```

Or for event-driven systems, define the key events:
```
order.placed   → triggers payment + inventory
payment.success → triggers notification + fulfillment
```

This forces you to think about what the system actually exposes before drawing boxes.

---

#### H — High-Level Design (10 min)

Draw the major components with approximate data flows:
- Clients → Load Balancer → App Servers → Cache → Database
- Async flows: App Server → Queue → Workers → Storage

Keep it high-level. Don't go into implementation details yet.

Say: *"Let me draw the happy path first, then we'll add complexity."*

---

#### A — Architecture Deep-Dive (10 min)

Pick **2–3 critical components** and go deep:
- The database choice and schema
- The caching strategy
- The queue and async processing flow
- The specific algorithm (geohash for Uber, consistent hashing for sharding, etc.)

Ask: *"Which component would you like to explore in more depth?"* — This invites the interviewer to direct you to what they care about.

---

#### D — Data Model (5 min)

Define your key schemas:
```
users:    { id, name, email, created_at }
tweets:   { id, user_id, content, created_at }
follows:  { follower_id, followee_id, created_at }
```

Mention your storage choice (SQL vs NoSQL) and justify it. Mention key indexes.

---

#### E — Edge Cases (5 min)

Proactively bring up failure modes:
- "What if the database goes down? We need read replicas."
- "What if a single user has 50M followers? Push fan-out would be too slow — we need a hybrid approach."
- "What if the queue backs up? We need DLQ (Dead Letter Queue) and alerting."

This is what distinguishes a senior from a mid-level candidate — **you think about failure before the interviewer asks**.

---

#### D — Deep Dive on Non-Functionals (7 min)

Address the non-functional requirements you captured in step 1:
- **Scalability**: How does each layer scale horizontally?
- **Reliability**: What are the single points of failure? How do we eliminate them?
- **Observability**: What metrics/logs/traces would you monitor?
- **Cost**: Are there expensive operations we can optimize? (e.g., avoid fan-out for celebrities)

---

### Clarifying Questions — The Best Ones

Questions that show senior thinking:

| Category | Question | Why It's Good |
|----------|----------|---------------|
| Scale | "Are we designing for 1M or 100M users?" | Changes every architecture decision |
| Read/Write ratio | "Is this read-heavy or write-heavy?" | Determines caching and DB strategy |
| Consistency | "Can we show slightly stale data (eventual) or must every read be fresh?" | Determines CAP choice |
| Latency | "What's the acceptable p99 latency for critical paths?" | Drives caching and sync/async decisions |
| Global | "Is this a single-region or multi-region system?" | Determines replication strategy |
| Features | "What's out of scope? (e.g., analytics, admin panel)" | Focuses the 45 minutes |

---

### Drawing Diagrams Effectively

**In whiteboard interviews:**
- Draw **boxes** (services) with **labels**
- Use **arrows** with labels (HTTP, WebSocket, async/Kafka, gRPC)
- Group related components (e.g., draw a box around "Database Layer")
- Leave space — you'll add components as you talk

**In virtual interviews:**
- Excalidraw, Miro, or Google Jamboard work well
- Use colors: blue for services, yellow for storage, green for clients, red for queues
- Keep it readable — an unreadable diagram is worse than a simple one

**What to draw first:**
1. Client → entry point (LB/API Gateway)
2. Core service(s)
3. Storage layer
4. Async flows (queues)
5. Add caching, CDN, monitoring as you discuss them

---

### What Service-Based Companies Expect (TCS, Infosys, Wipro, Capgemini, Cognizant)

**Depth required:** Medium. They test that you understand concepts and can apply them to real problems.

**Common topics:** Monolith vs microservices, REST API design, database choice, basic caching, load balancing, cloud basics (AWS/Azure).

**Tone:** Structured and confident. Show that you can lead a technical discussion.

**What makes you stand out:** Concrete examples from your work experience. "At Aziro, we used Redis for session management because..." — this is gold.

**Time in interview:** Often 20–30 minutes, combined with other topics. Don't over-engineer. A clean, simple design with justified trade-offs beats a complex one.

---

### What Product-Based Companies Expect (Swiggy, Zepto, PhonePe, Razorpay, Freshworks, Atlassian India)

**Depth required:** High. They test the same areas as FAANG but at a slightly reduced scope.

**Common topics:** All of Phases 1–6. Phase 7 topics for senior roles.

**Tone:** Collaborative. They want to see how you think, not just what you know. Think out loud.

**What makes you stand out:**
- Mentioning trade-offs proactively: "I chose eventual consistency here because the latency benefit outweighs the risk of stale feeds."
- Asking about their actual tech stack: "Does your team use Kafka already, or would this introduce new infrastructure?"
- Real operational experience: Your Docker, Kubernetes, CI/CD work is highly relevant.

**Time in interview:** 45–60 minutes, dedicated system design round. Use the full RESHADED framework.

---

### Common Mistakes and How to Avoid Them

| Mistake | Why It Hurts | Fix |
|---------|-------------|-----|
| Jumping to solution without requirements | Shows junior thinking | Always spend 5 min on requirements |
| Over-engineering from the start | Complexity without justification | Start simple, add complexity only when justified |
| Not justifying choices | "I'll use Kafka" — so what? | Always say "because..." |
| Ignoring non-functional requirements | Design without SLAs is incomplete | Always come back to latency, availability, scale |
| Getting stuck in one area | Misses the big picture | Set a timer mentally, keep moving |
| Silence | Interviewer can't assess your thinking | Think out loud, even if unsure |
| Saying "I don't know" and stopping | Missed opportunity | Say "I don't know the exact details, but my approach would be..." |
| Ignoring failure modes | Mid-level answer | Proactively mention failure scenarios |

---

### Tailoring Your MERN/AWS/Micro-Frontend Background

Your specific experience is an asset — use it explicitly:

**MERN Stack:**
- "I've worked extensively with MongoDB, so for document-based storage I'd use it here. For financial data, I'd switch to PostgreSQL for ACID guarantees."
- "In Node.js, async I/O means a single server can handle many concurrent connections, which is why it's well-suited for this API gateway layer."

**Micro-Frontend Architecture:**
- "This is similar to the micro-frontend pattern I've implemented — each domain team owns their slice of the UI and deploys independently. We'd apply the same principles on the backend."
- "We'd use Module Federation for frontend composition, and an API Gateway + BFF pattern to serve each frontend's specific data needs."

**AWS:**
- "For object storage, I'd use S3 with pre-signed URLs. For CDN, CloudFront in front of S3. For our compute, EC2 or ECS (containerized) behind an ALB."
- "I've set up GitHub Actions CI/CD pipelines deploying to EC2 with Nginx + PM2. For this system, we'd extend that to a Blue-Green deployment using CodeDeploy or Kubernetes rolling updates."

**AI/OpenAI API:**
- "I've integrated OpenAI's API including function calling and streaming responses. For this AI assistant feature, I'd design a RAG pipeline: user query → embed with `text-embedding-ada-002` → vector similarity search → top-K context → GPT-4 generation with context."

---

### Mock Interview Practice Guide

**Solo practice (first 2 weeks):**
1. Pick a case from [[Phase 6 - Real-World System Design Cases]]
2. Set a 45-minute timer
3. Speak out loud (or write) your design, following RESHADED
4. After 45 minutes, review against the notes and ByteByteGo's version
5. Identify gaps → review relevant phase

**Peer practice (essential):**
- Find a peer (ex-colleague, online — Pramp.com, interviewing.io, or Discord communities)
- Take turns as interviewer and interviewee
- Interviewer's job: ask follow-up questions, probe weak spots
- Give and receive honest feedback

**Self-recording:**
- Record yourself on your phone
- Watch it back — are you clear? Structured? Do you ramble?

**Target practice questions (in priority order):**
1. Design a notification system
2. Design a URL shortener
3. Design a Twitter-like feed
4. Design a ride-hailing service
5. Design a payment gateway
6. Design WhatsApp
7. Design a search autocomplete
8. Design Netflix/YouTube

---

## 📚 Suggested Resources

| Resource | Why |
|----------|-----|
| Pramp.com | Free peer mock interviews |
| interviewing.io | Paid mock interviews with engineers from top companies |
| ByteByteGo — *System Design Interview Vol 1 & 2* | Best structured practice cases |
| Gaurav Sen — "How to ace system design" (YouTube) | Interview-specific tips |
| Hello Interview — hellointerview.com | India + US focused system design prep |

---

## ✅ Progress Tracker

### Framework
- [ ] RESHADED framework memorized and internalized
- [ ] Can draw a clean high-level diagram in 10 minutes
- [ ] Have a set of go-to clarifying questions ready

### Company Calibration
- [ ] Understand service-based vs product-based depth expectations
- [ ] Know how to position MERN/AWS/AI experience in answers

### Practice
- [ ] Completed 5 solo timed design sessions (45 min each)
- [ ] Completed at least 2 peer mock interviews
- [ ] Watched at least 3 ByteByteGo case walkthroughs
- [ ] Identified and reviewed 3 personal weak areas

### Final Readiness Check
- [ ] Can do URL shortener, Twitter feed, chat system cleanly
- [ ] Can explain CAP theorem with examples in < 2 minutes
- [ ] Can justify SQL vs NoSQL choice for 3 different scenarios
- [ ] Can design a RAG system / AI product end-to-end
- [ ] Never silent for more than 30 seconds in a mock interview

---

> [!TIP]
> You're ready to interview when you can design a system you haven't seen before — not just the ones you studied. The framework + deep component knowledge + practice is what gets you there.

---

*You've completed the roadmap! 🎉 Go back to [[System Design Roadmap]] to review your progress.*
