---
tags: [system-design, phase-1, foundations, networking, dns, cdn]
status: not-started
phase: 1
---

# Phase 1 — Foundations

> [!NOTE]
> These are the "boring" infrastructure pieces that every system is built on. You won't be asked to design DNS — but you **will** be expected to know how it works when designing a global product.

---

## 🧭 Overview

Before designing systems, you need a working mental model of how the internet works end-to-end: how a request travels from a browser to a server and back, what happens at each layer, and where failure/latency can be introduced. This phase covers the foundational building blocks that every system design assumes you already know.

Given your Node.js/Express background, many of these will feel familiar — the goal is to go **one level deeper** than a developer needs to and understand the **trade-offs** from a system architect's perspective.

---

## 📋 Topics Table

| Topic | Level | Why It Matters | Est. Time |
|-------|-------|----------------|-----------|
| Client-Server model & request lifecycle | Must-Know | Mental model for all system design | 1 hr |
| DNS — how it works | Must-Know | Appears in global/multi-region designs | 1 hr |
| HTTP/HTTPS, HTTP/2, HTTP/3 | Must-Know | API design, performance, connection handling | 2 hrs |
| TCP vs UDP | Must-Know | Streaming, gaming, real-time systems | 1 hr |
| Load Balancers (L4 vs L7) | Must-Know | Every scalable system uses them | 2 hrs |
| CDN — Content Delivery Networks | Must-Know | Static assets, edge caching, latency | 1.5 hrs |
| Reverse Proxy vs Forward Proxy | Must-Know | Nginx, API gateways | 1 hr |
| Firewalls & basic network security | Good-to-Know | Enterprise/fintech designs | 1 hr |
| WebSockets & Long Polling | Must-Know | Real-time features (chat, notifications) | 2 hrs |
| REST vs GraphQL vs gRPC | Must-Know | API choice trade-offs | 2 hrs |

---

## 📖 Detailed Notes

### Client-Server Model & Request Lifecycle

A request from a browser to `api.myapp.com/todos`:
1. Browser checks local **DNS cache** → if miss, queries DNS resolver
2. DNS resolves to an IP (could be a CDN edge or load balancer IP)
3. **TCP handshake** (3-way: SYN → SYN-ACK → ACK) establishes connection
4. **TLS handshake** adds encryption overhead (~1–2 RTTs)
5. HTTP request sent → hits **Load Balancer** → routed to an **App Server**
6. App Server queries **Database** or **Cache**
7. Response travels back through the same path

> [!TIP] You likely know this — just revise
> You've built Express APIs that handle this flow. The interview upgrade is knowing where latency accumulates (DNS, TLS, DB) and how to reduce it (CDN, connection pooling, keep-alive).

---

### DNS — Domain Name System

DNS translates human-readable names to IPs. Key concepts for system design:

- **TTL (Time To Live)**: How long a DNS record is cached. Low TTL = faster failover, more DNS queries. High TTL = fewer queries, slower failover.
- **DNS-based load balancing**: Return different IPs for the same domain (Round Robin DNS). Simple, but no health-checking.
- **GeoDNS**: Return different IPs based on user's geographic location — used for routing to the nearest data center.
- **Record types**: A (IPv4), AAAA (IPv6), CNAME (alias), MX (mail), TXT (verification)

> [!NOTE]
> In multi-region designs (e.g., India + Singapore + US), GeoDNS is often the first layer of routing — before traffic even hits your load balancer.

---

### HTTP/HTTPS, HTTP/2, HTTP/3

| Version | Key Feature | Use Case |
|---------|-------------|----------|
| HTTP/1.1 | Persistent connections, pipelining | Legacy, still common |
| HTTP/2 | Multiplexing (multiple streams on 1 TCP conn), header compression | Modern REST APIs, browsers |
| HTTP/3 | Built on QUIC (UDP), eliminates TCP head-of-line blocking | Low-latency, mobile, streaming |

**Important for design:**
- HTTP/2 multiplexing reduces latency significantly for dashboards with many parallel API calls (relevant to your micro-frontend work)
- WebSockets upgrade from HTTP/1.1 — important for real-time systems
- Long-polling (Comet) is the fallback when WebSockets aren't available

---

### TCP vs UDP

| | TCP | UDP |
|--|-----|-----|
| Reliability | Guaranteed delivery, ordered | Best-effort, may drop packets |
| Speed | Slower (handshake + ACK) | Faster (no handshake) |
| Use cases | HTTP, databases, email | Video streaming, DNS, gaming, VoIP |

> [!TIP]
> In system design, mention UDP when designing: live video streaming, gaming leaderboards, DNS, or any system where dropping occasional packets is acceptable but latency must be minimal.

---

### Load Balancers — L4 vs L7

**Layer 4 (Transport layer):** Routes based on IP + port. Fast, but "dumb" — no knowledge of request content.

**Layer 7 (Application layer):** Routes based on HTTP headers, URL paths, cookies. Slower but smart.

| Feature | L4 LB | L7 LB |
|---------|--------|--------|
| Routing basis | IP + Port | URL, headers, cookies |
| SSL termination | No (or passthrough) | Yes |
| Content-based routing | ❌ | ✅ |
| Performance | Higher | Slightly lower |
| Example | AWS NLB | AWS ALB, Nginx |

**Load balancing algorithms:**
- **Round Robin** — requests distributed sequentially
- **Least Connections** — sent to server with fewest active connections
- **IP Hash** — same client always hits the same server (session stickiness)
- **Weighted Round Robin** — more powerful servers get more traffic

> [!TIP] You likely know this — just revise
> You've used Nginx. For interviews, be ready to discuss when you'd pick ALB vs NLB and why sticky sessions can cause hot spots.

**Health Checks:** Load balancers periodically ping servers. Unhealthy servers are removed from rotation. This is your basic availability mechanism.

---

### CDN — Content Delivery Networks

A CDN is a geographically distributed network of **edge servers** that cache and serve content close to users.

**How it works:**
1. User in Mumbai requests `cdn.myapp.com/logo.png`
2. CDN edge in Mumbai checks its cache
3. If miss → CDN fetches from **origin server** (your S3 / backend)
4. Cached at edge for future requests

**When to use CDN:**
- Static assets (JS, CSS, images, videos) → always
- API responses (with care, short TTLs) → for read-heavy endpoints
- DDoS protection (Cloudflare absorbs traffic at edge)

**Push vs Pull CDN:**
- **Pull**: CDN fetches from origin on first miss. Simpler. Used by CloudFront, Cloudflare.
- **Push**: You explicitly push content to CDN. Good for large files you know will be popular.

> [!TIP] You likely know this — just revise
> You've likely used AWS CloudFront or similar. For interviews, know the **cache invalidation problem** — CDN caching makes deployments tricky (use versioned filenames or cache-busting query params).

---

### Reverse Proxy vs Forward Proxy

**Forward Proxy:** Sits between client and internet. Client-side. Used for: anonymity, content filtering, corporate firewalls.

**Reverse Proxy:** Sits between internet and your servers. Server-side. Used for: load balancing, SSL termination, caching, rate limiting.

> [!NOTE]
> Nginx is almost always used as a reverse proxy in production. API Gateways (Kong, AWS API Gateway) are a specialized reverse proxy with added features (auth, rate limiting, routing).

---

### WebSockets & Long Polling

**Long Polling:** Client makes HTTP request → server holds it open until data is available → responds → client immediately reconnects. Simple fallback, high overhead.

**WebSockets:** Full-duplex connection over a single TCP connection. Established via HTTP upgrade. Best for: chat, live feeds, collaborative editing, real-time dashboards.

**Server-Sent Events (SSE):** Server pushes events to client over HTTP. One-directional. Good for: live notifications, live scores.

| | Long Polling | WebSockets | SSE |
|--|---|---|---|
| Direction | Bidirectional (via multiple requests) | Bidirectional | Server → Client only |
| Overhead | High | Low | Low |
| Proxy/firewall friendly | ✅ | ⚠️ (some issues) | ✅ |
| Use case | Fallback | Chat, gaming | Notifications, feeds |

---

### REST vs GraphQL vs gRPC

| | REST | GraphQL | gRPC |
|--|------|---------|------|
| Protocol | HTTP | HTTP | HTTP/2 |
| Data format | JSON | JSON | Protobuf (binary) |
| Over/under-fetching | Common problem | Solved | N/A |
| Schema | Implicit | Explicit (SDL) | Explicit (.proto) |
| Best for | Public APIs | Complex client needs | Internal microservices |
| Performance | Good | Moderate | Excellent |

> [!TIP] You likely know this — just revise
> As a MERN dev you've lived in REST. For interviews, be ready to say *why* you'd pick gRPC for internal microservice communication (typed contract, binary efficiency, streaming support).

---

## 📚 Suggested Resources

| Resource | Why |
|----------|-----|
| ByteByteGo — "How HTTPS works" (YouTube) | Visual, interview-focused |
| Gaurav Sen — "Load Balancing" (YouTube) | Clear L4 vs L7 explanation |
| ByteByteGo newsletter — "CDN explained" | Covers push/pull, invalidation |
| MDN Web Docs — HTTP overview | Authoritative HTTP/1.1 → HTTP/3 |
| *System Design Interview Vol 1* Ch. 1 | Scale from zero framework |

---

## ✅ Progress Tracker

- [ ] Client-server model and end-to-end request lifecycle
- [ ] DNS record types, TTL, GeoDNS
- [ ] HTTP/1.1 vs HTTP/2 vs HTTP/3 differences
- [ ] TCP vs UDP — when to use each
- [ ] Load balancer types (L4/L7), algorithms, health checks
- [ ] CDN — push vs pull, cache invalidation
- [ ] Reverse proxy vs forward proxy, API Gateway role
- [ ] WebSockets vs Long Polling vs SSE — pick the right one
- [ ] REST vs GraphQL vs gRPC trade-offs

---

*Next → [[Phase 2 - Core Building Blocks]]*
