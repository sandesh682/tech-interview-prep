# 🧠 09 — System Design with Node.js

---

📍 **Progress Marker**
- Last Revised: ___________
- Revision Count: 0
- Difficulty Level: 🧠 Expert

---

## Q1 — Design a scalable REST API backend that handles 100k requests/second

### ✅ Simple Explanation
At 100k RPS, a single Node.js server isn't enough. You need horizontal scaling, load balancing, caching, and DB optimization working together.

### 🧠 Deep Dive — Architecture Layers

```
Internet
    │
    ▼
[CDN - CloudFront/Cloudflare]   ← static assets, edge caching
    │
    ▼
[Load Balancer - Nginx/ALB]     ← distribute traffic, SSL termination
    │
    ├── [Node.js Instance 1] (PM2 cluster, 8 workers)
    ├── [Node.js Instance 2]
    ├── [Node.js Instance 3]
    └── [Node.js Instance N]
              │
              ▼
    [Redis Cluster]              ← sessions, rate limiting, caching, pub/sub
              │
              ▼
    [MongoDB Atlas / PostgreSQL] ← sharded or read replicas
              │
    [Message Queue - BullMQ/SQS] ← async job processing
```

**Key principles:**
- **Stateless servers**: sessions in Redis, not in-process memory
- **Cache aggressively**: Redis for hot data (user profiles, product listings)
- **Async for heavy tasks**: email sending, image processing → queue
- **Read replicas**: route read traffic to replicas, writes to primary
- **Horizontal pod autoscaling** (Kubernetes): scale based on CPU/RPS metrics

### 💻 Code Example
```js
// Redis-backed rate limiting (works across all instances)
const RedisStore = require('rate-limit-redis');
const rateLimit = require('express-rate-limit');
const { createClient } = require('redis');

const redisClient = createClient({ url: process.env.REDIS_URL });
await redisClient.connect();

const limiter = rateLimit({
  windowMs: 60 * 1000,
  max: 100,
  standardHeaders: true,
  store: new RedisStore({
    sendCommand: (...args) => redisClient.sendCommand(args),
  }),
});

// Cache-aside pattern with Redis
async function getProductWithCache(productId) {
  const cacheKey = `product:${productId}`;

  // Try cache first
  const cached = await redisClient.get(cacheKey);
  if (cached) return JSON.parse(cached);

  // Cache miss — fetch from DB
  const product = await Product.findById(productId).lean();
  if (!product) return null;

  // Store in cache with TTL
  await redisClient.setEx(cacheKey, 300, JSON.stringify(product)); // 5 min
  return product;
}

// Cache invalidation on update
async function updateProduct(id, data) {
  const product = await Product.findByIdAndUpdate(id, data, { new: true });
  await redisClient.del(`product:${id}`); // invalidate cache
  return product;
}
```

### ⚠️ Common Mistakes
- Using in-memory caching or sessions — breaks with multiple instances
- No cache invalidation strategy — stale data served indefinitely
- Sending emails/notifications synchronously in request handlers

### 🎯 Interview Tip
> "At scale, I move everything stateful (sessions, rate limits, locks) to Redis. The Node.js servers themselves stay stateless, which makes horizontal scaling trivial — just add more instances."

---

## Q2 — How do you implement a job queue system in Node.js?

### ✅ Simple Explanation
Use BullMQ (backed by Redis) to offload long-running or background tasks from request handlers. Producers add jobs; workers process them asynchronously.

### 🧠 Deep Dive
**When to use queues:**
- Sending emails/SMS
- Image/video processing
- Report generation
- Webhooks delivery (with retries)
- Scheduled tasks (cron)

**BullMQ features:**
- Job priorities
- Delayed jobs
- Retry with exponential backoff
- Rate-limited queues
- Job progress tracking
- Dead letter queue for failed jobs

### 💻 Code Example
```js
// Producer (in request handler)
const { Queue } = require('bullmq');

const emailQueue = new Queue('emails', {
  connection: { host: 'localhost', port: 6379 },
  defaultJobOptions: {
    attempts: 3,
    backoff: { type: 'exponential', delay: 1000 },
    removeOnComplete: 100,  // keep last 100 completed
    removeOnFail: 500,
  }
});

// Add job — returns immediately, doesn't wait for processing
app.post('/register', async (req, res) => {
  const user = await User.create(req.body);
  
  await emailQueue.add('welcome-email', {
    to: user.email,
    name: user.name,
    userId: user._id,
  });

  res.status(201).json({ user }); // responds fast, email sent async
});

// Worker (separate process)
const { Worker } = require('bullmq');
const nodemailer = require('nodemailer');

const transporter = nodemailer.createTransport({ /* config */ });

const emailWorker = new Worker('emails', async (job) => {
  const { to, name, userId } = job.data;

  await job.updateProgress(10);

  await transporter.sendMail({
    to,
    subject: `Welcome, ${name}!`,
    html: `<h1>Welcome aboard!</h1>`,
  });

  await job.updateProgress(100);
  return { sent: true, to };
}, {
  connection: { host: 'localhost', port: 6379 },
  concurrency: 5, // process 5 jobs in parallel
});

// Event handlers
emailWorker.on('completed', (job) => {
  console.log(`Email sent: job ${job.id}`);
});

emailWorker.on('failed', (job, err) => {
  console.error(`Email failed: job ${job.id}`, err.message);
});

// Scheduled job (cron-like)
await emailQueue.add('digest', { type: 'daily' }, {
  repeat: { cron: '0 9 * * *' }, // every day at 9am
});
```

### ⚠️ Common Mistakes
- Processing heavy jobs synchronously in request handlers — blocks event loop, high latency
- Not handling job failures — jobs disappear silently; use `removeOnFail: false` in dev

### 🎯 Interview Tip
> "BullMQ with Redis gives you durable queues, retries, and monitoring out of the box. In production, I run workers in separate PM2 processes so they don't share resources with the web server."

---

## Q3 — How do you implement real-time features in a Node.js app?

### ✅ Simple Explanation
Use WebSockets (via Socket.io) for bidirectional real-time communication — chat, notifications, live updates. For one-way server-to-client streaming, Server-Sent Events (SSE) are simpler.

### 🧠 Deep Dive
**Comparison:**

| Feature | WebSocket | SSE | Long Polling |
|---|---|---|---|
| Direction | Bidirectional | Server → Client | Server → Client |
| Protocol | WS/WSS | HTTP | HTTP |
| Reconnect | Manual | Automatic | Manual |
| Use case | Chat, gaming | Notifications, feeds | Legacy support |
| Load balancer | Sticky session OR Redis adapter | Normal | Normal |

**Scaling WebSockets:** Each user's socket connection lives on a specific server. With multiple servers, you need a Redis adapter so all servers share socket rooms.

### 💻 Code Example
```js
// Socket.io with Redis adapter (multi-server)
const { createServer } = require('http');
const { Server } = require('socket.io');
const { createAdapter } = require('@socket.io/redis-adapter');
const { createClient } = require('redis');

const httpServer = createServer(app);
const io = new Server(httpServer, {
  cors: { origin: process.env.CLIENT_URL, credentials: true },
  transports: ['websocket', 'polling'],
});

// Redis adapter — sync events across all Node.js instances
const pubClient = createClient({ url: process.env.REDIS_URL });
const subClient = pubClient.duplicate();
await Promise.all([pubClient.connect(), subClient.connect()]);
io.adapter(createAdapter(pubClient, subClient));

// Auth middleware for sockets
io.use(async (socket, next) => {
  try {
    const token = socket.handshake.auth.token;
    const payload = jwt.verify(token, process.env.JWT_ACCESS_SECRET);
    socket.userId = payload.sub;
    next();
  } catch {
    next(new Error('Authentication failed'));
  }
});

io.on('connection', (socket) => {
  console.log(`User ${socket.userId} connected`);

  // Join user's personal room
  socket.join(`user:${socket.userId}`);

  // Chat message
  socket.on('send-message', async ({ roomId, content }) => {
    const message = await Message.create({
      roomId, content, sender: socket.userId
    });
    // Emit to everyone in room (including sender)
    io.to(`room:${roomId}`).emit('new-message', message);
  });

  // Join chat room
  socket.on('join-room', (roomId) => {
    socket.join(`room:${roomId}`);
  });

  socket.on('disconnect', () => {
    console.log(`User ${socket.userId} disconnected`);
  });
});

// Send notification from REST endpoint (cross-server works via Redis adapter)
async function sendNotification(userId, notification) {
  await Notification.create({ userId, ...notification });
  io.to(`user:${userId}`).emit('notification', notification);
}

// SSE — simpler for one-way streams
app.get('/events', requireAuth, (req, res) => {
  res.setHeader('Content-Type', 'text/event-stream');
  res.setHeader('Cache-Control', 'no-cache');
  res.setHeader('Connection', 'keep-alive');

  const send = (data) => res.write(`data: ${JSON.stringify(data)}\n\n`);

  // Subscribe to Redis channel for this user
  const sub = redisClient.duplicate();
  sub.subscribe(`user:${req.user.id}:events`, (message) => {
    send(JSON.parse(message));
  });

  req.on('close', () => sub.quit());
});
```

### ⚠️ Common Mistakes
- Not using Redis adapter with Socket.io on multiple servers — events don't cross servers
- Not authenticating WebSocket connections — anyone can connect
- Not handling the `disconnect` event — leaked subscriptions

### 🎯 Interview Tip
> "Socket.io with Redis adapter makes WebSockets work transparently across multiple server instances. Without it, you'd need sticky sessions which complicates load balancing."

---

## Q4 — How do you design a Node.js microservices architecture?

### ✅ Simple Explanation
Break a monolith into independently deployable services, each owning its data. Services communicate via HTTP REST, gRPC, or async message queues.

### 🧠 Deep Dive
**Communication patterns:**
- **Synchronous (REST/gRPC)**: Direct service-to-service calls — use for queries needing immediate response
- **Asynchronous (Queue/Event Bus)**: Services emit events — use for decoupled workflows

```
[API Gateway]
    │
    ├──► [User Service]     ← owns users DB
    ├──► [Product Service]  ← owns products DB
    ├──► [Order Service]    ← owns orders DB
    └──► [Notification Service]

[Event Bus - RabbitMQ/Kafka/SQS]
    OrderService publishes "order.created"
    NotificationService subscribes → sends email
    InventoryService subscribes → updates stock
```

**Key patterns:**
- **API Gateway**: single entry point, auth, routing, rate limiting
- **Circuit Breaker**: fail fast if downstream service is down (Opossum library)
- **Saga pattern**: distributed transactions without 2PC
- **Service discovery**: how services find each other (Consul, Kubernetes DNS)

### 💻 Code Example
```js
// Circuit Breaker with Opossum
const CircuitBreaker = require('opossum');
const axios = require('axios');

function callUserService(userId) {
  return axios.get(`http://user-service/users/${userId}`, { timeout: 2000 });
}

const breaker = new CircuitBreaker(callUserService, {
  timeout: 3000,          // fail if takes > 3s
  errorThresholdPercentage: 50, // open if 50% fail
  resetTimeout: 30000,    // try again after 30s
});

breaker.fallback((userId) => ({ id: userId, name: 'Unknown', cached: true }));

breaker.on('open', () => console.error('Circuit OPEN — user-service down'));
breaker.on('close', () => console.log('Circuit CLOSED — user-service recovered'));

// Order service uses circuit breaker to call user service
app.post('/orders', requireAuth, async (req, res) => {
  try {
    const user = await breaker.fire(req.user.id);
    const order = await Order.create({ userId: user.id, ...req.body });

    // Async event — don't wait for notification
    await eventBus.publish('order.created', {
      orderId: order._id,
      userId: user.id,
      email: user.email,
      total: order.total,
    });

    res.status(201).json(order);
  } catch (err) {
    res.status(503).json({ error: 'Service temporarily unavailable' });
  }
});

// Health check endpoint (required for K8s liveness/readiness probes)
app.get('/health', async (req, res) => {
  const dbStatus = mongoose.connection.readyState === 1 ? 'ok' : 'error';
  const redisStatus = redisClient.isOpen ? 'ok' : 'error';

  const healthy = dbStatus === 'ok' && redisStatus === 'ok';

  res.status(healthy ? 200 : 503).json({
    status: healthy ? 'ok' : 'degraded',
    db: dbStatus,
    redis: redisStatus,
    uptime: process.uptime(),
  });
});
```

### ⚠️ Common Mistakes
- Sharing databases between services — defeats microservice isolation
- Synchronous chaining too many services — latency compounds (A → B → C → D)
- No circuit breaker — one slow downstream service cascades failures

### 🎯 Interview Tip
> "In microservices, I follow: one service, one database. Async events for inter-service communication where possible. Circuit breakers to prevent cascading failures. Health endpoints for orchestrators."

---

## Q5 — How do you implement caching strategies in Node.js?

### ✅ Simple Explanation
Cache expensive operations (DB queries, API calls, computation) at different levels: application cache (Redis), HTTP cache (ETags, Cache-Control), CDN.

### 🧠 Deep Dive
**Caching strategies:**
- **Cache-aside (Lazy)**: check cache → miss → load from DB → populate cache
- **Write-through**: write to cache AND DB simultaneously
- **Write-behind (Write-back)**: write to cache immediately, sync to DB async
- **Read-through**: cache handles DB read automatically

**Cache invalidation (the hard problem):**
1. TTL-based: auto-expire after N seconds
2. Event-based: invalidate on update/delete
3. Cache versioning: `product:v2:123` — change version to invalidate all

### 💻 Code Example
```js
// Multi-level cache: in-process (LRU) + Redis
const LRU = require('lru-cache');

// L1: In-process cache (microseconds, limited size)
const l1Cache = new LRU({
  max: 1000,
  ttl: 1000 * 60, // 1 min
});

// L2: Redis (milliseconds, shared across instances)
async function getWithCache(key, fetcher, ttlSeconds = 300) {
  // L1 hit
  const l1 = l1Cache.get(key);
  if (l1) return l1;

  // L2 hit
  const l2 = await redisClient.get(key);
  if (l2) {
    const parsed = JSON.parse(l2);
    l1Cache.set(key, parsed); // populate L1
    return parsed;
  }

  // Cache miss — fetch from source
  const data = await fetcher();
  if (data) {
    const serialized = JSON.stringify(data);
    await redisClient.setEx(key, ttlSeconds, serialized);
    l1Cache.set(key, data);
  }
  return data;
}

// HTTP Cache headers
app.get('/products/:id', async (req, res) => {
  const product = await getWithCache(
    `product:${req.params.id}`,
    () => Product.findById(req.params.id).lean(),
    600 // 10 min in Redis
  );

  if (!product) return res.status(404).json({ error: 'Not found' });

  // HTTP caching
  res.set({
    'Cache-Control': 'public, max-age=300', // 5 min browser cache
    'ETag': `"${product.updatedAt.getTime()}"`,
    'Last-Modified': product.updatedAt.toUTCString(),
  });

  // If client has current version, return 304
  if (req.headers['if-none-match'] === `"${product.updatedAt.getTime()}"`) {
    return res.status(304).send();
  }

  res.json(product);
});

// Cache invalidation on update
async function updateProduct(id, data) {
  const product = await Product.findByIdAndUpdate(id, data, { new: true });
  await redisClient.del(`product:${id}`);   // L2 invalidation
  l1Cache.delete(`product:${id}`);           // L1 invalidation
  return product;
}
```

### ⚠️ Common Mistakes
- Not setting TTL — cached data grows forever until Redis OOMs
- Caching user-specific data with a shared key — users see each other's data
- Over-caching — data becomes stale; financial/inventory data needs short TTLs or no caching

### 🎯 Interview Tip
> "I use a two-level cache — in-process LRU for the hottest data (sub-millisecond) and Redis for shared cache across instances. Cache keys always include version or user context to avoid cross-contamination."

---

## ⚡ Quick Revision Summary

- Scale-out: stateless Node.js + Redis for shared state + load balancer
- Cache-aside: check Redis → miss → DB → populate Redis → return
- Job queues: BullMQ for async/heavy work; workers in separate processes
- WebSockets + Redis adapter = multi-server real-time
- Microservices: one DB per service, circuit breakers, async events
- Health checks: `/health` endpoint required for K8s probes

---

## 🏆 Top 5 Must Revise Before Interview

1. **Cache-aside pattern** — the most common caching strategy
2. **Redis use cases** — sessions, rate limiting, pub/sub, cache, queues
3. **Circuit breaker pattern** — preventing cascading failures
4. **BullMQ job queue** — async processing architecture
5. **WebSocket scaling** — why Redis adapter is needed

---

## 🎤 Real Interview Questions

- *"Design a URL shortener that handles 10k RPS"*
- *"How would you scale your MERN app from 100 to 100k users?"*
- *"What is a circuit breaker? Why do you need one?"*
- *"How do you ensure a job queue doesn't lose jobs on server restart?"*
- *"Where would you add caching in a typical Express + MongoDB app?"*
