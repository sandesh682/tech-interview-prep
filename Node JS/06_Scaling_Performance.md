# 🔴 06 — Scaling & Performance

---

📍 **Progress Marker**
- Last Revised: ___________
- Revision Count: 0
- Difficulty Level: 🔴 Hard

---

## Q1 — How do you scale a Node.js application?

Scaling = **removing bottlenecks layer by layer**

---

## 1️⃣ Efficient Code (Foundation)

- Avoid blocking operations (sync code)
    
- Use async/await properly
    
- Use streams for large files
    
- Optimize DB queries (indexes, pagination)
    

👉 Node.js is single-threaded → bad code = bottleneck

**Problem:** Limited by single CPU core

---

## 2️⃣ Vertical Scaling (Scale Up)

- Increase CPU / RAM (bigger machine)
    

**Pros:**

- Quick & easy
    

**Cons:**

- Expensive
    
- Hard limit
    
- Single point of failure
    

---

## 3️⃣ Clustering (Multi-Core Usage)

- Use Node.js cluster module
    
- Run multiple processes on same machine
    

```js
const cluster = require('cluster');
const os = require('os');

if (cluster.isMaster) {
  os.cpus().forEach(() => cluster.fork());
} else {
  require('./server');
}
```

**Pros:**

- Uses all CPU cores
    

**Cons:**

- No shared memory
    
- Session issues
    

---

## 4️⃣ Load Balancer (Horizontal Scaling)

- Nginx / AWS ALB
    
- Distribute traffic across instances
    

**Pros:**

- Scalability
    
- Fault tolerance
    

**Cons:**

- Sticky sessions problem
    

---

## 5️⃣ Stateless Architecture

- Store sessions in Redis
    
- Store files in S3
    

**Tools:**

- Redis (cache/session)
    
- AWS S3 (storage)
    

**Pros:**

- Any server can handle any request
    

**Cons:**

- Added latency
    
- Infra complexity
    

---

## 6️⃣ Caching

- Cache DB queries (Redis)
    
- Use CDN (Cloudflare)
    

**Pros:**

- Huge performance boost
    
- Reduces DB load
    

**Cons:**

- Cache invalidation is hard
    

---

## 7️⃣ Background Jobs & Queues

- Use BullMQ / RabbitMQ
    

**Use cases:**

- Payment processing
    
- Email sending
    
- Image processing
    

**Pros:**

- Non-blocking APIs
    
- Better performance
    

**Cons:**

- Eventual consistency
    
- Retry complexity
    

---

## 8️⃣ Microservices (Advanced)

- Split into services:
    
    - Auth
        
    - Payments
        
    - Orders
        

**Pros:**

- Independent scaling
    
- Better for large teams
    

**Cons:**

- Distributed system complexity
    
- Network overhead
    

---

## 9️⃣ Docker (Containerization)

- Package app into containers
    

**Pros:**

- Same environment everywhere
    
- Easy deployment
    

**Cons:**

- Need orchestration for scale
    

---

## 🔟 Kubernetes (Orchestration)

- Manage containers at scale
    

**Features:**

- Auto-scaling
    
- Self-healing
    
- Rolling updates
    

**Cons:**

- Complex setup
    

---

## 1️⃣1️⃣ Monitoring & Observability

- Prometheus (metrics)
    
- Grafana (dashboards)
    

**Purpose:**

- Detect bottlenecks
    
- Debug production issues
    

---

## 🔥 Real-World Flow

Efficient Code  
→ Vertical Scaling  
→ Clustering  
→ Load Balancer  
→ Stateless (Redis + S3)  
→ Caching  
→ Queues  
→ Docker  
→ Kubernetes  
→ Microservices

---

## 🎯 Key Bottlenecks Mapping


---

## 🧩 Interview Insight

Scaling is NOT about tools.  
It’s about:  
👉 Identifying bottleneck  
👉 Applying correct solution

---
|Bottleneck Type|Problem Example|Solution|
|---|---|---|
|CPU (heavy computation)|Large loops, image processing|worker_threads|
|CPU (single-core limit)|Node using only 1 core|clustering|
|Traffic (high concurrency)|Too many users|Load Balancer|
|DB|Slow queries, repeated reads|Caching (Redis)|
|Slow APIs|Email, payments, uploads|Queues (BullMQ)|
|Deployment|Scaling infra|Docker/Kubernetes|
---

## Q2 — What are `worker_threads` and when should you use them?

`worker_threads` allow Node.js to run **JavaScript in parallel threads**, separate from the main event loop.

👉 Used to offload **CPU-heavy tasks**  
👉 Prevents blocking of the main thread

---

## ⚙️ Why worker_threads?

### Problem:

- Node.js is single-threaded
    
- CPU-heavy work blocks event loop
    

```js
for (let i = 0; i < 1e9; i++) {}
```

❌ Blocks:

- Incoming requests
    
- Entire server
    

---

### Solution:

👉 Move heavy work to worker threads

✔ Main thread stays responsive  
✔ CPU work runs in parallel

---

## 🚀 When to Use

### ✅ Use for:

- Image processing (resize, compression)
    
- Video processing (encoding, transcoding)
    
- Encryption / hashing
    
- Large JSON parsing
    
- Heavy computations (algorithms, loops)
    

---

### ❌ Do NOT use for:

- API calls
    
- Database queries
    
- File I/O
    
- Normal request handling
    

👉 Node handles I/O efficiently already

---

## 🧠 Real Production Flow

User Request  
→ API Server (Express)  
→ Queue (BullMQ)  
→ Worker Process  
→ Piscina Thread Pool  
→ worker_threads (CPU work)  
→ Result stored (DB/S3)  
→ User notified

---

## 💻 Example (Basic)

### main.js

```js
const { Worker } = require('worker_threads');

function runWorker() {
  return new Promise((resolve, reject) => {
    const worker = new Worker('./worker.js');

    worker.on('message', resolve);
    worker.on('error', reject);
  });
}
```

### worker.js

```js
const { parentPort } = require('worker_threads');

function heavyTask() {
  let sum = 0;
  for (let i = 0; i < 1e9; i++) {
    sum += i;
  }
  return sum;
}

parentPort.postMessage(heavyTask());
```

---

## ⚠️ Problems (If Used Incorrectly)

- Creating worker per request → crash ❌
    
- Too many threads → CPU contention ❌
    
- Large data transfer → performance hit ❌
    
- Complex debugging ❌
    

---

## ✅ Best Practices

### 1. Use Thread Pool (IMPORTANT)

- Use Piscina
    
- Avoid creating workers manually
    

---

### 2. Limit Threads

```js
maxThreads = CPU cores
```

---

### 3. Combine with Queue

- Use BullMQ
    
- Handle traffic spikes safely
    

---

### 4. Keep Workers Stateless

- Avoid shared mutable state
    
- Prevent race conditions
    

---

### 5. Use for CPU-bound ONLY

- Not for I/O tasks
    

---

## 🆚 worker_threads vs cluster

|Feature|worker_threads|cluster|
|---|---|---|
|Purpose|CPU-heavy tasks|Handle more requests|
|Type|Threads|Processes|
|Memory|Shared|Separate|

---

## 🎯 Key Insight

- worker_threads = parallel computation
    
- cluster = parallel servers
    

---

## 🔥 Real Production Use Case

### Example: Media Processing System

- User uploads image/video
    
- API pushes job to queue
    
- Worker picks job
    
- Piscina assigns thread
    
- worker_threads process:
    
    - Resize image
        
    - Compress video
        
- Result stored in S3
    
- User notified via polling/WebSocket
    

---

## 🧠 Mental Model

- API = receptionist 📞
    
- Queue = waiting line 🚶
    
- Piscina = manager 👨‍💼
    
- worker_threads = workers 🛠️
    

---

## 🧩 Final Takeaway

👉 Use worker_threads when:

- CPU is the bottleneck
    

👉 Avoid when:

- Problem is I/O or traffic
    

---
## Q3 — what is child_process (fork, spawn, exec)?

---

## 🧠 Core Idea

All three methods:  
👉 Create **separate OS-level processes**

But differ in:

- What they run
    
- How they communicate
    
- Performance & memory
    

---

# 🧩 fork()

## 🧠 What

- Creates a **new Node.js process**
    
- Runs a JS file
    

## ⚙️ Communication

- IPC (`process.send`, `on('message')`)
    

## ✅ Use When

- Need isolation (payments, critical jobs)
    
- Node.js background tasks
    

## ❌ Cons

- High memory
    
- Slower startup
    

---

# ⚙️ spawn()

## 🧠 What

- Runs an **external program** (Python, ffmpeg, etc.)
    
- No Node runtime
    

## ⚙️ Communication

- Streams (`stdout`, `stderr`)
    

## ✅ Use When

- Large data processing
    
- External tools / ML / video processing
    

## ❌ Cons

- Slightly complex API
    

---

# 🧪 exec()

## 🧠 What

- Runs a **shell command**
    
- Uses system shell (bash)
    

## ⚙️ Communication

- Buffered output (all at once)
    

## ✅ Use When

- Simple commands
    
- Small output tasks
    

## ❌ Cons

- Memory risk (buffers output)
    
- Slower (shell overhead)
    

---

# ⚔️ Key Differences

|Feature|fork()|spawn()|exec()|
|---|---|---|---|
|Process Type|Node.js process|External program|Shell process|
|Runs|JS file|Binary / script|Command|
|Communication|IPC|Streams|Buffered|
|Memory|High|Low|Medium|

---

# 🔍 Under the Hood

fork → New Node.js runtime (V8 + event loop)  
spawn → Direct program execution  
exec → Shell → command execution

---

# 🧠 When to Use

- fork() → Node task + isolation
    
- spawn() → External tools / large data
    
- exec() → Quick shell commands
    

---

# ⚠️ Performance Insight

- fork → heavy but safe
    
- spawn → efficient & scalable
    
- exec → simple but risky for large output
    

---

# 🎯 Key Insight

👉 All create processes, but:

- fork = Node-specific
    
- spawn = direct execution
    
- exec = shell wrapper
    

---

# 🧠 Mental Model

- fork → new Node server 🖥️
    
- spawn → run external program ⚙️
    
- exec → run shell command 🧪
    

---
## Q4 — How do you detect and fix memory leaks in Node.js?

## 📌 What is a Memory Leak?
A memory leak happens when objects are **not garbage collected** because references are still held, causing **memory usage to grow over time**.

---

## 🔍 Detection Techniques

### 1. Monitor Memory Usage
```js
setInterval(() => {
  console.log(process.memoryUsage());
}, 5000);
```

- `heapUsed` → key indicator  
- If it **keeps increasing continuously**, suspect a leak  

---

### 2. Use Chrome DevTools (Heap Snapshots)

- Run app:
```bash
node --inspect app.js
```

- Open: `chrome://inspect`
- Take **Heap Snapshots**
- Compare before & after load

👉 Look for:
- Growing objects
- Retained objects (not GC’d)

---

### 3. Use Profiling Tools

- `clinic.js` → performance + memory analysis  
- `heapdump` → capture heap snapshots  
- `--trace-gc` → observe garbage collection  

---

### 4. Reproduce Under Load

```bash
npx autocannon -c 50 -d 20 http://localhost:3000
```

👉 Helps confirm leak under real traffic  

---

## ⚠️ Common Causes of Memory Leaks

### ❌ Global Variables
```js
global.cache = [];
```

---

### ❌ Unremoved Event Listeners
```js
emitter.on("event", handler);
```

---

### ❌ Closures Holding Large Data
```js
function outer() {
  const big = new Array(1000000);
  return () => console.log(big);
}
```

---

### ❌ Unbounded Cache
```js
const cache = {};
cache[key] = largeObject;
```

---

### ❌ Uncleared Timers
```js
setInterval(() => {}, 1000);
```

---

### ❌ Large Buffers / Streams Not Released

---

## 🛠️ Fixing Memory Leaks

### ✅ Remove References
```js
data = null;
```

---

### ✅ Clean Event Listeners
```js
emitter.removeListener("event", handler);
// or
emitter.once()
```

---

### ✅ Use Bounded Cache (LRU)
```js
const LRU = require("lru-cache");

const cache = new LRU({
  max: 100,
  ttl: 1000 * 60 * 5
});
```

---

### ✅ Clear Timers
```js
clearInterval(intervalId);
```

---

### ✅ Avoid Globals
- Use `let` / `const`
- Enable `"use strict"`

---

### ✅ Use Streams for Large Data
```js
fs.createReadStream()
```

---

## 🧪 Real Debugging Workflow

1. **Monitor memory**
   - `process.memoryUsage()`

2. **Reproduce issue**
   - Load testing (autocannon)

3. **Capture heap snapshots**
   - Chrome DevTools

4. **Analyze retained objects**
   - Check dominators

5. **Fix root cause**
   - Remove references / optimize logic

6. **Verify**
   - Memory stabilizes after fix

---

## 📊 Healthy vs Leak Pattern

- ❌ Leak:
```
100MB → 150MB → 220MB → 300MB → 🚨
```

- ✅ Healthy:
```
100MB → 140MB → 130MB → 135MB → stable
```

---

## 🎯 Interview-Ready Answer

> Detect memory leaks by monitoring heap usage and taking heap snapshots using Chrome DevTools or tools like clinic.js. Reproduce the issue under load, identify retained objects, and fix common causes like unbounded caches, global variables, or unremoved listeners. Finally, verify that memory stabilizes.

---

## ⚡ Pro Tips

- Prefer **stateless design** when possible  
- Avoid long-lived in-memory storage  
- Always **limit cache size + TTL**  
- Monitor in production (APM tools)

---

## Q4 — What is connection pooling and why is it critical?

## 📌 Definition

Connection pooling is the practice of maintaining a **set of reusable database connections** instead of creating a new connection for every request.

👉 Instead of:

- Open → Query → Close ❌  
    We do:
    
- Reuse connection from pool → Query → Return to pool ✅
    

---

## ⚙️ How It Works

1. App starts → creates a pool (e.g., 10 connections)
    
2. Requests borrow a connection from the pool
    
3. Query executes
    
4. Connection is returned to pool (not closed)
    
5. If pool is full → requests go to **wait queue**
    

---

## 🚀 Why It Is Critical

### ⚡ Performance

- DB connection creation is expensive (TCP + authentication)
    
- Pooling removes repeated overhead → faster responses
    

### 🚀 Concurrency Control

- Limits number of active DB connections
    
- Prevents DB overload under high traffic
    

### 🛑 Resource Protection

- Databases have connection limits
    
- Pool ensures safe usage within limits
    

### ⏱️ Low Latency

- Reusing connections reduces response time
    

### 📈 Scalability

- Handle more users with fewer resources
    

---

## 🧠 Core Concept (VERY IMPORTANT)

### 🧩 Pool + Queue System

- Pool size = **fixed (e.g., 10)**
    
- Extra requests → **wait queue**
    

#### Example:

- 100 requests come
    
    - 10 execute immediately
        
    - 90 wait in queue
        

---

## ⏱️ Queue Behavior (Critical Insight)

### Default

- Queue is handled internally by MongoDB driver (no manual setup needed)
    

### Configurable Behavior

#### 🔑 `waitQueueTimeoutMS`

```js
waitQueueTimeoutMS: 5000
```

- Request waits **max 5 seconds** for connection
    
- If not available → ❌ timeout error
    

👉 Without this:

- Requests may wait indefinitely (bad UX)
    

---

## ⚙️ MongoDB (Mongoose) Configuration

Using Mongoose

```js
const mongoose = require("mongoose");

mongoose.connect(uri, {
  maxPoolSize: 10,              // max concurrent connections
  minPoolSize: 2,               // keep some ready
  waitQueueTimeoutMS: 5000,     // fail if waiting too long
  serverSelectionTimeoutMS: 5000
});
```

---

## 🚨 What Happens Under Load

### Scenario:

- Pool size = 10
    
- 100 requests arrive
    

### Behavior:

1. 10 requests execute immediately
    
2. 90 requests wait in queue
    
3. If connection frees within 5 sec → request proceeds ✅
    
4. If not → ❌ timeout error
    

---

## ⚠️ Common Mistakes

### ❌ Creating connections per request

```js
// WRONG
await mongoose.connect(...)
```

### ✅ Correct

- Connect once at app startup
    

---

### ❌ Ignoring queue timeout

- Leads to hanging requests
    

---

### ❌ Too small pool size

- Causes long wait queue
    

---

### ❌ Too large pool size

- Can overload DB
    

---

## 📊 Pool Sizing Strategy

- Small apps → 5–10
    
- Medium apps → 10–20
    
- High traffic → tune based on:
    
    - DB capacity
        
    - Query time
        
    - CPU cores
        

---

## 🔥 Advanced Insights (Interview Gold)

### 🧠 Fail-Fast Strategy

- Using `waitQueueTimeoutMS` avoids infinite waiting
    
- Improves system reliability
    

---

### 🔁 Backpressure Handling

- Queue acts as natural backpressure mechanism
    
- Protects DB from spikes
    

---

### 🚨 Pool Exhaustion Symptoms

- High latency
    
- Timeout errors
    
- Growing request queue
    

---

### 🛠️ When You Need More Than Pooling

- Rate limiting (API level)
    
- Retry logic
    
- Circuit breaker (for resilience)
    

---

## 🎯 Interview One-Liner

Connection pooling is the reuse of a fixed number of pre-established database connections to improve performance, control concurrency, and prevent database overload, while excess requests are queued and handled efficiently.

---

## 🧩 Simple Analogy

- Pool = 10 checkout counters 🛒
    
- 100 customers arrive
    
    - 10 served immediately
        
    - 90 wait in line
        
    - If waiting too long → customer leaves (timeout)
        

---

## ✅ Summary

- Pool = **performance + stability**
    
- Queue = **handles overflow**
    
- Timeout = **prevents hanging**
    
- Proper tuning = **production readiness**
    

---

## Q5 — How do you profile and optimize a slow Node.js API?

## 🧠 Step 1: Profile First (Don’t Guess)

### 🔍 Tools

- `node --inspect` + Chrome DevTools (CPU, memory, heap)
    
- Clinic.js (Doctor / Flame)
    
- APM:
    
    - New Relic
        
    - Datadog
        

### 📊 What to Check

- CPU usage
    
- Memory usage / heap growth
    
- Event loop lag
    
- DB query time
    
- External API latency
    

---

## 🔥 Step 2: Identify Cause → Apply Fix

---

### 🧠 1. CPU Issue

**Symptoms**

- High CPU, slow under load
    

**Causes**

- Heavy computation, sync code
    

**Fix**

- Worker threads / queues
    
- Optimize algorithms
    

---

### 🧠 2. Memory Issue

**Symptoms**

- Increasing memory, crashes
    

**Causes**

- Memory leaks, large objects
    

**Fix**

- Heap snapshots
    
- Clear references, limit cache
    

---

### ⏳ 3. Event Loop Blocking

**Symptoms**

- Requests delayed, high latency
    

**Causes**

- Sync APIs, long loops
    

**Fix**

- Use async APIs
    
- Streams for large data
    

---

### 🗄️ 4. Database Bottleneck

**Symptoms**

- Slow queries
    

**Causes**

- Missing indexes, N+1 queries
    

**Fix**

- Indexing, query optimization
    
- Connection pooling
    

---

### 🌐 5. External API Latency

**Symptoms**

- Slow due to third-party calls
    

**Causes**

- Sequential requests
    

**Fix**

- `Promise.all`, timeouts, retries
    
- Caching
    

---

### 📦 6. Payload Size Issue

**Symptoms**

- Slow network response
    

**Causes**

- Large JSON
    

**Fix**

- Send minimal data
    
- Compression, pagination
    

---

### 🔄 7. Concurrency Issue

**Symptoms**

- Slow under high traffic
    

**Causes**

- Single-thread limits
    

**Fix**

- Clustering, load balancing
    
- Background jobs
    

---

### 🔌 8. Connection Pool Exhaustion

**Symptoms**

- Requests waiting / timing out
    

**Causes**

- Small pool, slow queries
    

**Fix**

- Tune pool size
    
- Use `waitQueueTimeoutMS`
    

---

## 🎯 Interview One-Liner

Profile using tools to identify whether the bottleneck is CPU, memory, event loop, DB, or I/O, then apply targeted optimizations like async processing, query tuning, caching, and scaling.

---
## Q6 — ⚙️ Manual Clustering vs PM2 Cluster Mode

---

## 📌 Core Idea

Node.js is **single-threaded**, so clustering is used to utilize **multiple CPU cores**.

---

## 🚀 PM2 Cluster Mode (Default Choice)

Using PM2:

```bash
pm2 start app.js -i max
```

### ✅ Benefits

- Multi-core usage (automatic)
    
- Built-in load balancing
    
- Auto-restart on crash
    
- Zero-downtime deploy (`pm2 reload`)
    
- Simple & production-ready
    

👉 **Best for most applications**

---

## 🤔 Why Not Always PM2?

Because it abstracts everything — sometimes you need **more control**

---

## ⚙️ Manual Clustering (Node.js `cluster` module)

### 🧠 When to Use

#### 🎛️ Fine-Grained Control

- Custom worker lifecycle (create/kill/restart)
    
- Custom load balancing logic
    

---

#### 🔌 Special Use-Cases

- Sticky sessions (WebSockets)
    
- Worker-specific roles (cron, background jobs)
    
- Inter-process communication (IPC)
    

---

#### 🏗️ Custom Systems

- Building frameworks / infra
    
- Highly optimized backend systems
    

---

#### ☁️ Container Environments

- Docker / Kubernetes
    
- Scaling handled externally → PM2 often unnecessary
    

---

## ⚠️ Trade-offs

### ❌ Manual Clustering

- Complex to manage
    
- No built-in zero-downtime
    
- Error-prone
    

### ✅ PM2

- Simple
    
- Reliable
    
- Production standard
    

---

## 🎯 When to Use What

- Normal backend → ✅ PM2
    
- Advanced/custom needs → ⚙️ Manual clustering
    
- Kubernetes → 🚀 Scale containers instead
    

---

## 🎯 Interview One-Liner

Manual clustering is used for fine-grained control and special use cases like sticky sessions or custom worker management, but in most production scenarios, PM2 cluster mode is preferred for simplicity, reliability, and zero-downtime deployments.

---

# Q6 — Node.js Scaling — Clustering vs PM2 vs Docker vs Kubernetes (Clear Mental Model)

---

## 🎯 Core Problem

> How do we use CPU efficiently and scale Node.js apps?

There are **2 types of scaling**:

- **Vertical scaling** → Use multiple CPU cores in same machine
    
- **Horizontal scaling** → Use multiple machines/containers
    

---

## 🧩 1. Node.js Clustering (Manual)

### 📌 What it does

- Uses Node `cluster` module
    
- Spawns workers = CPU cores
    

```js
const cluster = require('cluster');
const os = require('os');

if (cluster.isPrimary) {
  for (let i = 0; i < os.cpus().length; i++) cluster.fork();
}
```

### ✅ Use when

- Single VM
    
- No Docker/Kubernetes
    

### ❌ Problem

- Limited to 1 machine
    
- Manual management
    

---

## ⚙️ 2. PM2 Cluster Mode

Using PM2

```bash
pm2 start app.js -i max
```

### 📌 What it does

- Same as clustering but automated
    
- Adds restart, logs, monitoring
    

### ✅ Use when

- VM-based deployment
    
- Want quick production setup
    

### ❌ Problem

- Still single machine
    
- Not cloud-native
    

---

## 🐳 3. Docker (Containers)

### 📌 What it does

- Packages app
    
- Runs isolated containers
    

```bash
docker run my-app
```

### 🧠 Important behavior

#### Case A: No CPU limit

- Machine: 8 cores
    
- Container sees: 8 cores
    
- Cluster → 8 workers  
    👉 Uses all cores (shared with others)
    

#### Case B: CPU limit set

```bash
docker run --cpus="2" my-app
```

- Container limited to 2 CPUs
    
- But cluster may still spawn 8 workers ❌  
    👉 Leads to CPU contention
    

---

### ✅ Best Practice with Docker

> “1 container = 1 Node.js process”

👉 Scale using **multiple containers**, NOT clustering

---

## ☸️ 4. Kubernetes (Orchestration)

Using Kubernetes

```yaml
replicas: 5
```

### 📌 What it does

- Manages containers (pods)
    
- Handles:
    
    - Auto-scaling
        
    - Load balancing
        
    - Self-healing
        

---

## ⚠️ Critical Mistake (Over-Scaling)

### ❌ Example:

- Node cluster → 8 workers
    
- Docker → 2 containers
    
- Kubernetes → 5 replicas
    

👉 Total processes = 8 × 2 × 5 = **80 workers**

👉 Machine cores = 8

👉 Result:

- CPU contention
    
- Context switching
    
- Poor performance
    

---

## ✅ Correct Design

```text
Kubernetes
   ↓
Pods (replicas)
   ↓
Container
   ↓
Node.js (1 process)
```

👉 Example:

- 8 cores machine
    
- 8 pods  
    👉 Each gets ~1 core → optimal
    

---

## ⚙️ Docker + PM2

### Option 1: Without PM2 (Recommended ✅)

```dockerfile
CMD ["node", "app.js"]
```

👉 Use Docker restart policy:

```bash
--restart=always
```

---

### Option 2: With PM2

```dockerfile
CMD ["pm2-runtime", "app.js"]
```

### 🧠 PM2 gives:

- Restart
    
- Logs
    
- Monitoring
    

### ❌ But:

- Docker already does most of this
    
- Adds unnecessary layer
    

---

## ❓ Do we need PM2?

|Setup|PM2 Needed?|
|---|---|
|VM only|✅ Yes|
|Docker only|⚠️ Optional|
|Docker + Kubernetes|❌ No|

---

## 🔥 Final Rules (Very Important)

### 🟢 Rule 1

> “1 container = 1 Node.js process”

---

### 🟢 Rule 2

> “Avoid clustering inside containers”

---

### 🟢 Rule 3

> “Use Kubernetes for scaling, not PM2/cluster”

---

### 🟢 Rule 4

> “Don’t mix scaling layers blindly”

---

## 🧠 Mental Model

|Tool|Responsibility|
|---|---|
|Cluster / PM2|Use CPU inside machine|
|Docker|Package + isolate|
|Kubernetes|Scale + manage system|

---

## ⚖️ Final Comparison

|Approach|Scaling|Best Use Case|
|---|---|---|
|Manual Cluster|Vertical|Learning / low-level|
|PM2|Vertical|Simple VM production|
|Docker|Horizontal|Medium-scale apps|
|Kubernetes|Horizontal + Auto|Large-scale systems|

---

## 🎯 Interview Answers

### Q: Docker + PM2 needed?

> “Not required. Docker can manage process lifecycle. We usually run one Node process per container.”

---

### Q: Kubernetes + PM2?

> “No. Kubernetes replaces PM2 responsibilities like restart, scaling, and load balancing.”

---

### Q: Clustering vs Kubernetes?

> “Clustering is vertical scaling on one machine, while Kubernetes provides horizontal scaling across multiple containers and nodes.”

---

## 🏁 Final Conclusion

👉 Small app → PM2 on VM  
👉 Growing app → Docker (no PM2)  
👉 Large-scale system → Kubernetes (no PM2, no clustering)

---

## 💡 One-Line Summary

> “Use clustering only on single machines; in containerized environments, scale with containers—not processes.”

---

## ⚡ Quick Revision Summary

- Cluster: one process per CPU core, shared port — scales I/O bound servers
- Worker threads: parallel JS execution, shared memory — for CPU-bound work
- Memory leaks: global refs, listener leaks, unclosed streams, unbounded caches
- Connection pooling: reuse DB connections; `release()` after every use or pool exhausts
- Profile with `clinic.js` or `--inspect`; common fixes: indexes, caching, streaming
- PM2 cluster mode is production standard; zero-downtime with `pm2 reload`

---

## 🏆 Top 5 Must Revise Before Interview

1. **Cluster vs worker_threads** — clear use case distinction
2. **Memory leak causes** — and how to detect them
3. **Connection pool exhaustion** — `release()` pattern, sizing formula
4. **Event loop blocking detection** — tools and mitigation
5. **PM2 cluster mode** — shows production awareness

---

## 🎤 Real Interview Questions

- *"Node is single-threaded. How do you use all 8 cores on a server?"*
- *"What would cause a Node.js process to grow in memory indefinitely?"*
- *"When would you use `worker_threads` vs the cluster module?"*
- *"Our API slows down under load after ~1000 requests. What would you check?"*
