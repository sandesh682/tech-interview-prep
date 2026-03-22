---
tags: [system-design, phase-7, advanced, ml-systems, real-time, search, openai]
status: not-started
phase: 7
---

# Phase 7 — Advanced & Specialized Topics

> [!NOTE]
> Not every topic here will come up in every interview. Prioritize based on your target company. **Product-based** companies (Swiggy, Razorpay, Freshworks) care more about real-time and data pipeline topics. Companies building AI products care about ML system design. Service-based company interviews rarely go this deep.

---

## 🧭 Overview

This phase covers specialized domains that distinguish truly senior candidates: designing ML-serving systems, real-time data pipelines, distributed search, observability infrastructure, and AI/LLM integrations. Your OpenAI API experience gives you a meaningful head start on the AI systems section.

---

## 📋 Topics Table

| Topic | Level | Why It Matters | Est. Time |
|-------|-------|----------------|-----------|
| Designing ML systems / feature stores | Must-Know | Senior roles at product companies | 2 hrs |
| Real-time data pipelines (Kafka + Spark/Flink) | Must-Know | Analytics, monitoring, fraud detection | 2 hrs |
| Distributed search (Elasticsearch at scale) | Good-to-Know | Search features in any product | 1.5 hrs |
| Observability: logging, metrics, tracing | Must-Know | Production readiness, SRE topics | 1.5 hrs |
| AI/LLM system design (RAG, agents) | Must-Know | Your competitive edge — be very strong here | 2.5 hrs |
| Content moderation at scale | Good-to-Know | Social platforms, user-generated content | 1 hr |
| Multi-tenancy patterns | Good-to-Know | SaaS architecture (Freshworks, Zoho) | 1.5 hrs |
| Global / multi-region architecture | Must-Know | Latency, data residency, failover | 1.5 hrs |
| Infrastructure as Code & CI/CD patterns | Good-to-Know | Your DevOps work is relevant | 1 hr |
| Designing for compliance (GDPR, PCI-DSS) | Good-to-Know | Fintech, healthtech, enterprise | 1 hr |

---

## 📖 Detailed Notes

### Designing ML Systems & Feature Stores

**ML system components:**
1. **Data pipeline** — collect, clean, transform raw data for training
2. **Feature Store** — centralized repository of features (pre-computed inputs for ML models)
   - Offline store (for training): S3, data warehouse
   - Online store (for inference): Redis, DynamoDB (low-latency lookup)
3. **Model Training** — periodic batch jobs (Spark, SageMaker)
4. **Model Registry** — version and track trained models (MLflow, SageMaker Model Registry)
5. **Model Serving** — serve predictions in real-time (REST API wrapping the model)
6. **Monitoring** — detect model drift (when real-world data diverges from training data)

**Feature store — why it matters:**
Without a feature store: each team re-computes the same features differently → inconsistency between training and serving (training-serving skew).
With a feature store: features computed once, stored, reused across models.

**Recommendation System design (common interview case):**
- **Candidate generation**: Retrieve K candidate items (collaborative filtering, ANN search on embeddings)
- **Ranking**: Score each candidate with a more expensive model
- **Filtering**: Remove already-seen items, apply business rules
- Serve via a low-latency inference API backed by Redis for pre-computed scores

---

### Real-Time Data Pipelines

**Lambda Architecture:**
- **Batch layer**: Processes all historical data, recomputes views periodically (Hadoop/Spark)
- **Speed layer**: Processes recent data in real-time (Kafka Streams / Flink)
- **Serving layer**: Merges batch + speed layer results for queries

**Kappa Architecture:**
- Simplification: only a stream processing layer (Kafka + Flink)
- Replay historical data through the same stream pipeline if needed
- Preferred in modern systems — less operational complexity

**Apache Flink vs Spark Streaming:**
| | Flink | Spark Streaming |
|--|-------|-----------------|
| Processing model | True streaming | Micro-batch |
| Latency | Very low (ms) | Low (seconds) |
| State management | Built-in, excellent | External state |
| Use case | Real-time alerts, fraud | Analytics, ETL |

**Common pipeline patterns:**
- **Fraud detection**: Transaction events → Kafka → Flink (compute velocity features) → ML model → block/allow decision in < 100ms
- **Analytics dashboard**: User events → Kafka → ClickHouse → Grafana (real-time metrics)
- **ETL**: Raw events → Kafka → Spark → data warehouse (Redshift/BigQuery) → BI tools

---

### Distributed Search (Elasticsearch at Scale)

**Elasticsearch internals:**
- **Index**: Logical collection of documents
- **Shard**: Unit of distribution. Each index is split into N primary shards (+ replica shards).
- **Inverted index**: Term → list of document IDs containing that term
- **Segment**: Immutable sub-unit of a shard. New documents go to new segments; old segments are merged.

**Scaling Elasticsearch:**
- Add shards for write throughput (more shards = more parallel indexing)
- Add replicas for read throughput and availability
- Dedicated master nodes (not data nodes) for cluster stability
- Hot-warm architecture: recent data on fast SSDs (hot nodes), older data on cheap HDDs (warm nodes)

**Near real-time indexing:**
- Documents are available for search within ~1 second of indexing (refresh interval)
- Behind the scenes: written to in-memory buffer → periodically flushed to a new segment

**Sync strategy with your primary DB:**
- Change Data Capture (CDC) via Debezium → publishes DB changes as events → Kafka → Elasticsearch consumer
- Or: Application writes to both DB and ES (simpler, but risks inconsistency on failure → use Outbox pattern)

---

### Observability: Logging, Metrics, Tracing

The three pillars of production observability:

**Logging:** Record discrete events.
- Structured logs (JSON) → searchable in ELK stack (Elasticsearch + Logstash + Kibana) or CloudWatch Logs
- Log levels: ERROR > WARN > INFO > DEBUG
- Never log sensitive data (passwords, PII, card numbers)

**Metrics:** Numeric measurements over time.
- Counters (total requests), Gauges (current memory), Histograms (request latency distribution)
- Prometheus collects metrics → Grafana visualizes → AlertManager sends alerts
- Key metrics to define: p50, p95, p99 latency (not just average)

**Distributed Tracing:** Track a request as it flows through multiple services.
- Each request gets a `traceId` and each service call gets a `spanId`
- Traces collected by Jaeger or Zipkin, or AWS X-Ray
- Critical for debugging latency in microservices ("which service added the 2-second delay?")

> [!TIP] You likely know this — just revise
> In your AWS/Node.js projects you've used CloudWatch. For interviews, mention the three pillars together and know what tool serves which purpose.

**SLO monitoring:**
- Define error budget: if SLO is 99.9% uptime, you have 8.7 hrs/year downtime budget
- Alert when error budget is burning too fast
- Runbooks: documented steps for responding to each alert

---

### AI/LLM System Design

> [!IMPORTANT]
> This is your strongest differentiator given your OpenAI API experience. Go deep here — it's a rare skill among senior SDE candidates in India's job market.

**Core LLM integration patterns:**

**1. RAG (Retrieval-Augmented Generation):**
The most common pattern for building AI products on top of LLMs without fine-tuning.

*How it works:*
1. Index your documents/knowledge base into a **vector database** (Pinecone, Weaviate, pgvector, Qdrant)
2. User asks a question → embed the question using the same embedding model
3. Vector similarity search retrieves the top-K relevant chunks
4. Retrieved context + user question sent to LLM as a prompt
5. LLM generates a grounded response

*System components:*
- **Document ingestion pipeline**: Chunk documents → embed → store in vector DB
- **Embedding model**: OpenAI `text-embedding-ada-002` or open-source alternatives
- **Vector DB**: Pinecone (managed), pgvector (PostgreSQL extension), Qdrant (self-hosted)
- **LLM API**: OpenAI GPT-4, Anthropic Claude, or self-hosted (Ollama)
- **Response cache**: Cache embedding + query results for identical/similar queries

**2. LLM Agents & Tool Use:**
LLMs given access to tools (function calling) can take actions: search the web, query a DB, call APIs.

*Design considerations:*
- **Tool definition**: Define available tools with JSON schema (function name, description, parameters)
- **Retry logic**: LLM may call a tool incorrectly → validate output, retry with error context
- **Rate limiting**: LLM API calls are expensive → rate limit per user, cache responses
- **Streaming**: Stream tokens to client for perceived low latency (`response.pipe(res)`)
- **Context window management**: Long conversations hit token limits → summarize history, use memory layers

**3. Prompt Management:**
- Store prompts in version-controlled config (not hardcoded)
- A/B test prompt variations
- Log prompts + responses for quality evaluation

**Cost optimization:**
- Cache responses for identical queries (semantic caching with vector similarity)
- Use cheaper models for simple tasks, expensive models for complex reasoning
- Batch API calls where possible
- Compress context: summarize chat history instead of sending full history

**Latency optimization:**
- Stream responses (don't wait for full generation)
- Pre-compute embeddings for known documents
- Use edge-deployed smaller models for latency-critical paths

> [!TIP] You likely know this — just revise
> Position your OpenAI API integration work precisely: "I've built RAG pipelines using OpenAI embeddings and integrated function calling for tool-use agents. The key production challenges I've solved are context window management, response caching, and cost optimization via semantic caching."

---

### Multi-Tenancy Patterns

For SaaS products (Freshworks, Zoho, Chargebee), multi-tenancy is foundational.

**Isolation models:**

| Model | Isolation | Cost | Complexity |
|-------|-----------|------|------------|
| **Silo** (one DB per tenant) | Highest | Highest | High |
| **Pool** (shared DB, shared schema, tenant_id column) | Lowest | Lowest | Low |
| **Bridge** (shared DB, separate schema per tenant) | Medium | Medium | Medium |

**Pool model (most common for mid-market SaaS):**
- Every table has a `tenant_id` column
- All queries must include `WHERE tenant_id = ?` (row-level security)
- Risk: tenant data leakage if `tenant_id` filter is missed
- Mitigation: RLS (Row-Level Security) in PostgreSQL — enforced at DB level

**Noisy neighbor problem:** One large tenant's workload impacts others. Mitigation: per-tenant rate limiting, dedicated resources for large tenants.

---

### Global / Multi-Region Architecture

For systems serving users across geographies (India + SEA + US):

**Active-Active:** Multiple regions, all serving traffic and accepting writes. Requires cross-region replication and conflict resolution. Complex.

**Active-Passive:** One primary region (active), others are hot standbys. Failover on primary failure. Simpler, but cross-region latency for passive region users.

**Data residency:** Regulations (GDPR, India's DPDP Act) may require user data to be stored in specific countries. Design storage partitioned by user's country.

**GeoDNS:** Route users to the nearest regional entry point.

**Global load balancing:** Anycast routing (AWS Global Accelerator) routes users to nearest PoP.

**Read replicas in each region:** Low-latency reads for users worldwide. Writes go to primary region → replicated to others.

---

### Infrastructure as Code & CI/CD Patterns

> [!TIP] You likely know this — just revise
> Your GitHub Actions + EC2 + Docker work is directly relevant. Articulate it at a system level.

**IaC tools:**
- **Terraform**: Declarative, cloud-agnostic. Define infrastructure as `.tf` files. Used by most companies.
- **AWS CDK / CloudFormation**: AWS-native. CDK lets you write IaC in TypeScript/Python.

**CI/CD pipeline stages:**
```
Code Push → Lint + Test → Build Docker Image → Push to Registry 
→ Deploy to Staging → Integration Tests → Deploy to Production (blue-green or rolling)
```

**Deployment strategies:**
- **Rolling**: Replace old instances gradually. Zero downtime but mixed versions during rollout.
- **Blue-Green**: Two environments (blue = current, green = new). Switch DNS/LB when green is ready. Instant rollback by switching back.
- **Canary**: Route 5% of traffic to new version, monitor, then gradually increase.

---

## 📚 Suggested Resources

| Resource | Why |
|----------|-----|
| ByteByteGo — "ML System Design" (YouTube) | Feature stores, recommendation systems |
| Chip Huyen — *Designing Machine Learning Systems* (book) | Best ML system design book |
| OpenAI Cookbook — cookbook.openai.com | RAG, function calling patterns (you know this) |
| LangChain documentation | Agent + RAG patterns |
| Martin Fowler — "Multi-tenancy" articles | SaaS architecture |
| Prometheus/Grafana documentation | Metrics and alerting setup |

---

## ✅ Progress Tracker

- [ ] ML system components: feature store, model serving, drift monitoring
- [ ] RAG pipeline — can design end-to-end (vector DB, chunking, retrieval)
- [ ] LLM agent design — tool use, retry, context window management
- [ ] LLM cost + latency optimization techniques
- [ ] Real-time data pipeline: Kafka + Flink/Spark — Lambda vs Kappa architecture
- [ ] Elasticsearch sharding, hot-warm architecture, CDC sync
- [ ] Three pillars of observability — logging, metrics, tracing
- [ ] SLO / error budget concept
- [ ] Multi-tenancy: pool vs silo vs bridge + noisy neighbor
- [ ] Active-Active vs Active-Passive multi-region
- [ ] Blue-Green vs Canary deployment strategies

---

*Next → [[Phase 8 - Interview Strategy & Practice]]*
