# 🟠 03 — Event Loop Deep Dive

> This is the #1 topic asked in senior Node.js interviews. Master this completely.

---

📍 **Progress Marker**
- Last Revised: ___________
- Revision Count: 0
- Difficulty Level: 🟠 Hard

---

## Q1 — Explain the Node.js Event Loop in detail

### ✅ Simple Explanation
The Event Loop is what allows Node.js to perform non-blocking I/O operations despite being single-threaded. It continuously picks tasks from queues and executes them.

### 🧠 Deep Dive — The 6 Phases (libuv)

```
   ┌──────────────────────────────┐
┌─>│           timers             │  ← setTimeout, setInterval callbacks
│  └──────────────┬───────────────┘
│  ┌──────────────▼───────────────┐
│  │     pending callbacks        │  ← I/O errors deferred from prev tick
│  └──────────────┬───────────────┘
│  ┌──────────────▼───────────────┐
│  │       idle, prepare          │  ← internal use only
│  └──────────────┬───────────────┘
│  ┌──────────────▼───────────────┐
│  │           poll               │  ← fetch new I/O events; execute callbacks
│  └──────────────┬───────────────┘
│  ┌──────────────▼───────────────┐
│  │           check              │  ← setImmediate callbacks
│  └──────────────┬───────────────┘
│  ┌──────────────▼───────────────┐
└──┤      close callbacks         │  ← socket.on('close'), etc.
   └──────────────────────────────┘
```

**Between EACH phase** (and between each task in a phase), Node drains:
1. **`process.nextTick` queue** (highest priority — runs before everything)
2. **Microtask queue** (Promise callbacks — `.then`, `.catch`, `queueMicrotask`)

### 💻 Code Example — Execution Order
```js
console.log('1: sync');

setTimeout(() => console.log('5: setTimeout'), 0);

setImmediate(() => console.log('4: setImmediate'));

Promise.resolve().then(() => console.log('3: promise'));

process.nextTick(() => console.log('2: nextTick'));

console.log('1b: sync end');

// Output:
// 1: sync
// 1b: sync end
// 2: nextTick        ← nextTick queue drained first
// 3: promise         ← microtask queue
// 4: setImmediate    ← check phase
// 5: setTimeout      ← timers phase (next loop iteration)
```

### ⚠️ Common Mistakes
- Thinking `setTimeout(fn, 0)` is instant — it's NOT; minimum ~1ms, runs in next timers phase
- Thinking `setImmediate` runs before `setTimeout(fn, 0)` always — **only guaranteed inside I/O callbacks**
- Not knowing `process.nextTick` runs BEFORE Promises

### 🎯 Interview Tip
> "After each phase, and after each task within a phase, Node.js drains the nextTick queue first, then the microtask queue. This is why nextTick has higher priority than Promises."

---

## Q2 — `process.nextTick` vs `setImmediate` vs `setTimeout(fn, 0)`

### ✅ Simple Explanation
All three defer execution, but they run at different points in the event loop.

### 🧠 Deep Dive

| API | Queue | When it runs | Priority |
|---|---|---|---|
| `process.nextTick` | nextTick queue | Before I/O, after current op | Highest |
| `Promise.then` | microtask queue | After nextTick queue | 2nd |
| `setImmediate` | check phase | After poll phase (I/O) | 3rd |
| `setTimeout(fn,0)` | timers phase | Next event loop iteration | 4th |

**`nextTick` starvation risk:** If you keep calling `process.nextTick` recursively, the event loop never advances — I/O is starved!

### 💻 Code Example
```js
// ✅ nextTick: emit event after current constructor finishes
class MyEmitter extends EventEmitter {
  constructor() {
    super();
    // Can't emit in constructor — no listeners yet
    process.nextTick(() => this.emit('ready'));
  }
}
const emitter = new MyEmitter();
emitter.on('ready', () => console.log('Ready!')); // Will fire

// ⚠️ Starvation example — DON'T DO THIS
function starveLoop() {
  process.nextTick(starveLoop); // I/O never runs!
}

// setImmediate vs setTimeout(0) inside I/O callback
const fs = require('fs');
fs.readFile('./file.txt', () => {
  setTimeout(() => console.log('setTimeout'), 0);
  setImmediate(() => console.log('setImmediate')); // ALWAYS first inside I/O
});

// Nested nextTick + Promise ordering
process.nextTick(() => console.log('nextTick 1'));
Promise.resolve().then(() => {
  console.log('promise 1');
  process.nextTick(() => console.log('nextTick inside promise')); // runs before promise 2!
});
Promise.resolve().then(() => console.log('promise 2'));
// Output: nextTick 1 → promise 1 → nextTick inside promise → promise 2
```

### ⚠️ Common Mistakes
- Thinking `setImmediate` is like `setTimeout(fn, 0)` — it runs in a completely different phase
- Not knowing that `nextTick` callbacks inside a Promise `.then` still run before the next `.then`

### 🎯 Interview Tip
> "Inside an I/O callback, `setImmediate` will always run before `setTimeout(fn, 0)`. Outside I/O, the order is non-deterministic depending on OS timer resolution."

---

## Q3 — What is libuv and what is the thread pool?

### ✅ Simple Explanation
`libuv` is the C library that powers Node.js's event loop. It also provides a thread pool for handling operations that can't be done asynchronously by the OS.

### 🧠 Deep Dive
**What libuv handles:**
- Event loop coordination
- Thread pool (default: 4 threads, max 1024)
- Async network I/O (via OS: `epoll` on Linux, `kqueue` on macOS)
- Async file I/O, DNS, crypto

**Thread pool is used for:**
- `fs` module (file operations)
- `crypto` (hashing, pbkdf2, randomBytes for large ops)
- `dns.lookup()` (but NOT `dns.resolve()`)
- `zlib` (compression)
- `http.request` (DNS resolution part)

**Network I/O does NOT use thread pool** — it uses OS-level async (epoll/kqueue).

### 💻 Code Example
```js
// Increase thread pool size (before any I/O)
process.env.UV_THREADPOOL_SIZE = 16; // set in code or as env var

// Example: 4 concurrent crypto operations
// With default pool size = 4, these run in parallel
// The 5th would queue until one finishes
const crypto = require('crypto');
function hashPassword(password) {
  return new Promise((resolve, reject) => {
    crypto.pbkdf2(password, 'salt', 100000, 64, 'sha512', (err, key) => {
      if (err) reject(err);
      else resolve(key.toString('hex'));
    });
  });
}

// Benchmark: increasing UV_THREADPOOL_SIZE helps when you have
// many concurrent CPU-bound async ops (crypto, zlib)
console.time('4 hashes');
await Promise.all([
  hashPassword('pass1'),
  hashPassword('pass2'),
  hashPassword('pass3'),
  hashPassword('pass4'),
]);
console.timeEnd('4 hashes');
```

### ⚠️ Common Mistakes
- Thinking ALL async operations use the thread pool — network I/O doesn't
- Not knowing that `dns.lookup()` uses thread pool (synchronous under the hood in libuv) but `dns.resolve()` is truly async

### 🎯 Interview Tip
> "Node.js is non-blocking for network I/O (handled by OS), but it uses libuv's thread pool for file I/O and crypto. The default pool size is 4 — increasing it helps for CPU-bound async tasks like bcrypt hashing."

---

## Q4 — What blocks the event loop and how do you fix it?

### ✅ Simple Explanation
The event loop is blocked when a synchronous operation takes too long — no other callbacks can run during this time. This tanks throughput in production.

### 🧠 Deep Dive
**Common blocking operations:**
- `JSON.parse/stringify` on very large objects
- Regex with catastrophic backtracking
- `fs.readFileSync`, `fs.writeFileSync`
- Synchronous crypto operations
- Long-running loops

**Detection tools:**
- `--inspect` + Chrome DevTools
- `blocked-at` npm package
- `clinic.js` (flamegraph, bubbleprof)
- `perf_hooks` for measuring latency

**Fix strategies:**
1. Use async equivalents (`fs.readFile` instead of `fs.readFileSync`)
2. Break up heavy computation with `setImmediate` (yield control)
3. Move CPU work to `worker_threads`
4. Use streaming for large data

### 💻 Code Example
```js
// ❌ Blocks event loop for large arrays
function findPrime(max) {
  // ... heavy computation, blocks for seconds
}
app.get('/prime', (req, res) => {
  const result = findPrime(10000000); // BAD
  res.json({ result });
});

// ✅ Yield control with setImmediate (chunked processing)
async function processChunked(items, chunkSize = 1000) {
  const results = [];
  for (let i = 0; i < items.length; i += chunkSize) {
    const chunk = items.slice(i, i + chunkSize);
    results.push(...chunk.map(processItem));
    await new Promise(resolve => setImmediate(resolve)); // yield to event loop
  }
  return results;
}

// ✅ Use worker_threads for CPU-heavy work
const { Worker } = require('worker_threads');
app.get('/prime', (req, res) => {
  const worker = new Worker('./prime-worker.js', {
    workerData: { max: 10000000 }
  });
  worker.on('message', result => res.json({ result }));
  worker.on('error', err => res.status(500).json({ error: err.message }));
});

// Measure event loop lag
const { monitorEventLoopDelay } = require('perf_hooks');
const h = monitorEventLoopDelay({ resolution: 20 });
h.enable();
setInterval(() => {
  console.log('Mean loop delay:', h.mean / 1e6, 'ms');
}, 1000);
```

### ⚠️ Common Mistakes
- Using `fs.readFileSync` in request handlers — fine in startup, terrible in hot paths
- Running `JSON.parse` on MB-sized payloads in a tight loop
- Not profiling before optimizing — measure first

### 🎯 Interview Tip
> "I profile with `clinic.js` or `--inspect` to find blocking operations. For CPU-heavy tasks, I use `worker_threads`. For chunked processing, I yield with `setImmediate` to let I/O through."

---

## Q5 — What is the difference between microtasks and macrotasks?

### ✅ Simple Explanation
- **Macrotasks** (task queue): `setTimeout`, `setInterval`, `setImmediate`, I/O callbacks
- **Microtasks**: `Promise.then`, `process.nextTick`, `queueMicrotask`

Microtasks are processed completely after EACH macrotask — before the next macrotask runs.

### 🧠 Deep Dive
```
Macrotask → [drain all microtasks] → Macrotask → [drain all microtasks] → ...
```

**Priority order:**
1. Current synchronous code runs to completion
2. `process.nextTick` queue drained (all of them, including newly added)
3. Microtask queue drained (Promises — including newly added)
4. Next macrotask runs
5. Repeat

This means: if a microtask adds another microtask, it runs before any macrotask.

### 💻 Code Example
```js
// Execution order challenge
setTimeout(() => console.log('A: setTimeout'), 0);

Promise.resolve()
  .then(() => {
    console.log('B: promise 1');
    return Promise.resolve();
  })
  .then(() => console.log('C: promise 2'));

queueMicrotask(() => console.log('D: queueMicrotask'));

process.nextTick(() => console.log('E: nextTick'));

console.log('F: sync');

// Output:
// F: sync
// E: nextTick
// B: promise 1
// D: queueMicrotask
// C: promise 2
// A: setTimeout
```

### ⚠️ Common Mistakes
- Not knowing that `queueMicrotask` is equivalent to `Promise.resolve().then()` but without creating a Promise object
- Thinking microtasks and nextTick are the same (nextTick always runs before Promises)

### 🎯 Interview Tip
> "Memorize this order: sync → nextTick → microtasks (Promises) → macrotasks (setTimeout/setImmediate/I/O). This is the most common interview execution-order puzzle."

---

## ⚡ Quick Revision Summary

- Event loop has 6 phases: timers → pending callbacks → idle → poll → check → close
- Between each phase: nextTick queue → microtask queue → next phase
- `nextTick` > `Promise.then` > `setImmediate` > `setTimeout(fn,0)`
- libuv thread pool (default 4) handles: fs, crypto, dns.lookup, zlib
- Network I/O does NOT use thread pool — uses OS async (epoll/kqueue)
- Event loop blocking: use `worker_threads` for CPU work, async APIs for I/O
- Microtasks drain completely before next macrotask

---

## 🏆 Top 5 Must Revise Before Interview

1. **Event loop phase diagram** — draw it from memory
2. **Execution order puzzle** — nextTick vs Promise vs setTimeout
3. **setImmediate vs setTimeout(0)** — inside vs outside I/O callbacks
4. **Thread pool** — what uses it, how to increase UV_THREADPOOL_SIZE
5. **Event loop blocking** — what causes it, how to fix it

---

## 🎤 Real Interview Questions

- *"What is the order of execution: nextTick, Promise.then, setImmediate, setTimeout?"*
- *"Why might Node.js not be suitable for CPU-intensive tasks?"*
- *"What is the poll phase of the event loop?"*
- *"How would you detect and fix an event loop block in production?"*
- *"Does `dns.lookup` use the thread pool? What about `dns.resolve`?"*
