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

### ✅ Simple Explanation
Memory leaks occur when your application holds references to objects that should have been garbage collected. Over time, memory grows and the process eventually crashes or slows down.

### 🧠 Deep Dive
**Common causes:**
1. **Global variables** — never garbage collected
2. **Event listener leaks** — not removing listeners (EventEmitter)
3. **Closure leaks** — closures holding large scopes longer than needed
4. **Timers** — `setInterval` callbacks holding references
5. **Cache without eviction** — unbounded in-memory maps/objects
6. **Database connections** — not releasing pool connections
7. **Streams** — not destroying on error

**Detection tools:**
- `process.memoryUsage()` — baseline monitoring
- `--inspect` + Chrome DevTools Memory tab
- `heapdump` npm package
- `clinic.js` memory doctor
- `node --max-old-space-size=512` — set heap limit to expose leaks earlier

### 💻 Code Example
```js
// ❌ Global variable leak
const cache = {};
app.get('/data/:id', async (req, res) => {
  cache[req.params.id] = await fetchData(req.params.id); // grows forever
  res.json(cache[req.params.id]);
});

// ✅ Use LRU cache with eviction
const LRU = require('lru-cache');
const cache = new LRU({ max: 500, ttl: 1000 * 60 * 5 }); // 500 items, 5 min TTL

// ❌ Event listener leak
function setupSocket(socket) {
  process.on('SIGTERM', () => {
    socket.destroy(); // adds listener every time, never removed!
  });
}
// ✅ Fix
function setupSocket(socket) {
  const handler = () => socket.destroy();
  process.once('SIGTERM', handler); // once, or remove manually
}

// ❌ Interval holding reference to large object
const bigData = new Array(1_000_000).fill('x');
setInterval(() => {
  // bigData is captured in closure — never GC'd
  console.log(bigData.length);
}, 1000);

// Memory usage monitoring
setInterval(() => {
  const { heapUsed, heapTotal } = process.memoryUsage();
  console.log(`Heap: ${(heapUsed / 1024 / 1024).toFixed(2)} MB / ${(heapTotal / 1024 / 1024).toFixed(2)} MB`);
}, 5000);

// Detect listener leak
const EventEmitter = require('events');
EventEmitter.defaultMaxListeners = 15; // default is 10, logs warning if exceeded
```

### ⚠️ Common Mistakes
- Not removing event listeners (`removeListener` / `off`) after use
- Using `Map`/`Set` as cache without eviction policy
- Not destroying readable streams after error — they hold file handles

### 🎯 Interview Tip
> "I track heap usage in production metrics. When I see a sawtooth pattern (heap grows, then drops at restart), that's a classic leak. I use Chrome DevTools heap snapshots to find which object types are growing."

---

## Q4 — What is connection pooling and why is it critical?

### ✅ Simple Explanation
A connection pool maintains a set of reusable database connections. Instead of opening/closing a connection per query (expensive), you borrow from the pool and return it when done.

### 🧠 Deep Dive
**Without pooling:**
- TCP handshake per query → ~50-100ms latency overhead
- Under load: thousands of connections saturate the DB

**Pool internals:**
- `min`: connections kept alive always
- `max`: maximum concurrent connections
- `acquire timeout`: how long to wait if pool is exhausted
- `idle timeout`: close connections idle too long

**Bottleneck:** Pool size should match DB's max connections / number of app instances.

### 💻 Code Example
```js
// Mongoose connection pooling (MongoDB)
mongoose.connect(process.env.MONGODB_URI, {
  maxPoolSize: 10,       // max concurrent connections
  minPoolSize: 2,        // keep at least 2 alive
  maxIdleTimeMS: 30000,  // close idle connections after 30s
  serverSelectionTimeoutMS: 5000,
  socketTimeoutMS: 45000,
});

// pg (PostgreSQL) pool
const { Pool } = require('pg');
const pool = new Pool({
  host: process.env.DB_HOST,
  database: process.env.DB_NAME,
  max: 20,                      // max connections
  idleTimeoutMillis: 30000,
  connectionTimeoutMillis: 2000, // fail fast if pool exhausted
});

// Always release connections!
async function getUser(id) {
  const client = await pool.connect();
  try {
    const result = await client.query('SELECT * FROM users WHERE id = $1', [id]);
    return result.rows[0];
  } finally {
    client.release(); // CRITICAL — if missing, pool exhausts
  }
}

// Better: use pool.query directly (auto-releases)
async function getUser2(id) {
  const result = await pool.query('SELECT * FROM users WHERE id = $1', [id]);
  return result.rows[0];
}

// Pool monitoring
pool.on('connect', () => console.log('New DB connection created'));
pool.on('error', (err) => console.error('Idle client error', err));
```

### ⚠️ Common Mistakes
- Forgetting to call `client.release()` in a try/catch — pool exhausts and all requests hang
- Setting pool size too high — DB gets overwhelmed (DB max_connections ÷ app instances = per-instance pool size)

### 🎯 Interview Tip
> "Pool size formula: `(DB max connections) / (number of app instances)`. Too high and you overwhelm the DB. Too low and requests queue. With Mongoose I set maxPoolSize around 10 for a typical service."

---

## Q5 — How do you profile and optimize a slow Node.js API?

### ✅ Simple Explanation
Profile first, optimize second. Use Node's built-in profiler or clinic.js to find bottlenecks — don't guess.

### 🧠 Deep Dive
**Profiling tools:**
- `node --prof app.js` → generates V8 profile → `node --prof-process` to analyze
- `clinic.js` (flame, bubbleprof, doctor) — visual profiling
- `--inspect` + Chrome DevTools Performance tab
- `autocannon` / `k6` — load testing to find breaking point

**Common optimizations:**
- Add DB indexes for slow queries
- Cache frequent reads (Redis)
- Stream large responses
- Compress responses (`compression` middleware)
- Avoid N+1 queries (use aggregation or JOINs)
- Use HTTP/2 for multiple parallel requests
- Set proper `keepAlive` on HTTP agents

### 💻 Code Example
```js
// Response compression
const compression = require('compression');
app.use(compression({
  filter: (req, res) => {
    if (req.headers['x-no-compression']) return false;
    return compression.filter(req, res);
  },
  level: 6, // 1-9, trade CPU for compression ratio
}));

// HTTP Keep-Alive (important for outbound requests)
const http = require('http');
const https = require('https');

const httpAgent = new http.Agent({ keepAlive: true });
const httpsAgent = new https.Agent({ keepAlive: true });

// Use with axios
const axios = require('axios');
const api = axios.create({
  baseURL: 'https://api.example.com',
  httpAgent,
  httpsAgent,
  timeout: 5000,
});

// Cache expensive operations with Redis
const redis = require('ioredis');
const redisClient = new redis(process.env.REDIS_URL);

async function getCachedUser(id) {
  const cached = await redisClient.get(`user:${id}`);
  if (cached) return JSON.parse(cached);

  const user = await User.findById(id);
  await redisClient.setex(`user:${id}`, 300, JSON.stringify(user)); // TTL 5 min
  return user;
}

// Load test with autocannon
// npx autocannon -c 100 -d 30 http://localhost:3000/api/todos
```

### ⚠️ Common Mistakes
- Enabling compression for already-compressed content (images, videos) — wastes CPU
- Not using `keepAlive` for outbound HTTP connections — TCP handshake per request overhead

### 🎯 Interview Tip
> "My optimization process: load test → profile → identify bottleneck (usually DB query or missing cache) → fix → repeat. Never optimize without measuring first."

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
