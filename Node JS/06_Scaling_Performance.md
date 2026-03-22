# 🔴 06 — Scaling & Performance

---

📍 **Progress Marker**
- Last Revised: ___________
- Revision Count: 0
- Difficulty Level: 🔴 Hard

---

## Q1 — How do you scale a Node.js application?

### ✅ Simple Explanation
Scale Node.js horizontally: run multiple processes (one per CPU core) to utilize all available cores, since one Node.js process only uses one core.

### 🧠 Deep Dive
**Scaling strategies:**
1. **Cluster module** — multiple processes, shared TCP port, built-in
2. **PM2** — process manager with clustering, zero-downtime restarts
3. **Horizontal scaling** — multiple machines behind a load balancer
4. **Vertical scaling** — more RAM/CPU (limited benefit for Node)

**Cluster vs worker_threads:**
| Feature | cluster | worker_threads |
|---|---|---|
| Process type | Separate OS processes | Threads in same process |
| Memory isolation | ✅ Full | ❌ Shared memory |
| Use case | Scaling I/O servers | CPU-bound computation |
| IPC | Via `process.send` (JSON) | SharedArrayBuffer / MessageChannel |
| Crash isolation | ✅ Worker crash doesn't kill master | ❌ Worker crash can kill process |

### 💻 Code Example
```js
// cluster module
const cluster = require('cluster');
const os = require('os');
const express = require('express');

const NUM_WORKERS = os.cpus().length;

if (cluster.isPrimary) {
  console.log(`Primary ${process.pid} started. Forking ${NUM_WORKERS} workers`);

  for (let i = 0; i < NUM_WORKERS; i++) {
    cluster.fork();
  }

  // Auto-restart crashed workers
  cluster.on('exit', (worker, code, signal) => {
    console.log(`Worker ${worker.pid} died. Restarting...`);
    cluster.fork();
  });

} else {
  // Worker process — each runs its own Express server
  const app = express();

  app.get('/health', (req, res) => {
    res.json({ pid: process.pid, status: 'ok' });
  });

  app.listen(3000, () => {
    console.log(`Worker ${process.pid} listening on port 3000`);
  });
}
```

```bash
# PM2 — preferred in production
pm2 start app.js -i max        # cluster mode, all CPUs
pm2 start app.js -i 4          # 4 workers
pm2 reload app.js              # zero-downtime restart
pm2 logs                       # view logs
pm2 monit                      # monitoring dashboard
```

### ⚠️ Common Mistakes
- Using in-memory session storage with cluster — different worker handles each request, sessions don't sync. Use Redis.
- Using cluster for CPU-heavy tasks — workers still block the event loop; use `worker_threads` instead

### 🎯 Interview Tip
> "In production, I use PM2 in cluster mode. For CPU tasks within a request, I offload to `worker_threads`. Cluster is for scaling I/O-bound servers across cores."

---

## Q2 — What are `worker_threads` and when should you use them?

### ✅ Simple Explanation
`worker_threads` run JavaScript in parallel threads within the same process, sharing memory via `SharedArrayBuffer`. Use them for CPU-intensive operations that would block the event loop.

### 🧠 Deep Dive
**When to use worker_threads:**
- Image processing / resizing
- PDF generation
- Complex math / ML inference
- Parsing massive JSON/XML
- Video transcoding

**Communication:**
- `parentPort.postMessage()` — structured clone (copies data)
- `SharedArrayBuffer` — zero-copy, but requires Atomics for sync
- `MessageChannel` — direct channel between two workers

**Worker pool pattern:** Spawn N workers at startup, queue jobs — avoids spawn overhead per request.

### 💻 Code Example
```js
// main.js
const { Worker, isMainThread, parentPort, workerData } = require('worker_threads');
const path = require('path');

function runInWorker(data) {
  return new Promise((resolve, reject) => {
    const worker = new Worker(path.join(__dirname, 'cpu-worker.js'), {
      workerData: data
    });
    worker.on('message', resolve);
    worker.on('error', reject);
    worker.on('exit', (code) => {
      if (code !== 0) reject(new Error(`Worker exited with code ${code}`));
    });
  });
}

app.get('/compute', async (req, res) => {
  const result = await runInWorker({ n: 1_000_000 });
  res.json({ result });
});

// cpu-worker.js
const { parentPort, workerData } = require('worker_threads');

function heavyComputation(n) {
  let sum = 0;
  for (let i = 0; i < n; i++) sum += i;
  return sum;
}

const result = heavyComputation(workerData.n);
parentPort.postMessage(result);

// Worker Pool implementation
class WorkerPool {
  constructor(workerFile, numWorkers = os.cpus().length) {
    this.workers = [];
    this.queue = [];

    for (let i = 0; i < numWorkers; i++) {
      this._addWorker(workerFile);
    }
  }

  _addWorker(file) {
    const worker = new Worker(file);
    worker.idle = true;
    worker.on('message', (result) => {
      worker.currentResolve(result);
      worker.idle = true;
      this._processQueue();
    });
    this.workers.push(worker);
  }

  _processQueue() {
    if (!this.queue.length) return;
    const idleWorker = this.workers.find(w => w.idle);
    if (!idleWorker) return;

    const { data, resolve, reject } = this.queue.shift();
    idleWorker.idle = false;
    idleWorker.currentResolve = resolve;
    idleWorker.postMessage(data);
  }

  run(data) {
    return new Promise((resolve, reject) => {
      this.queue.push({ data, resolve, reject });
      this._processQueue();
    });
  }
}
```

### ⚠️ Common Mistakes
- Spawning a new Worker per request — expensive, defeats the purpose. Use a pool.
- Sharing mutable state via `SharedArrayBuffer` without `Atomics` — race conditions

### 🎯 Interview Tip
> "I always use a worker pool rather than spawning workers per request — it avoids the overhead of thread creation and OS-level memory allocation."

---

## Q3 — How do you detect and fix memory leaks in Node.js?

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
