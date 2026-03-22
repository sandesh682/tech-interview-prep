# 🟡 02 — Core Concepts: Async, Promises, Callbacks

---

📍 **Progress Marker**
- Last Revised: ___________
- Revision Count: 0
- Difficulty Level: 🟡 Medium

---

## Q1 — Callback Hell — what is it and how do you fix it?

### ✅ Simple Explanation
Callback hell (pyramid of doom) happens when callbacks are nested inside callbacks, making code hard to read, debug, and maintain.

### 🧠 Deep Dive
Root cause: Node.js was originally callback-first. Each async operation needs a callback, and sequential async ops create deep nesting.

**Evolution of async patterns:**
1. Callbacks → error-first convention `(err, data) => {}`
2. Promises → `.then().catch()` chaining
3. async/await → syntactic sugar over Promises

### 💻 Code Example
```js
// ❌ Callback Hell
fs.readFile('a.txt', (err, dataA) => {
  if (err) return handleError(err);
  fs.readFile('b.txt', (err, dataB) => {
    if (err) return handleError(err);
    db.save({ dataA, dataB }, (err, result) => {
      if (err) return handleError(err);
      console.log('Saved:', result);
    });
  });
});

// ✅ Promisified version
const readFile = util.promisify(fs.readFile);

async function processFiles() {
  try {
    const [dataA, dataB] = await Promise.all([
      readFile('a.txt'),
      readFile('b.txt')
    ]);
    const result = await db.save({ dataA, dataB });
    console.log('Saved:', result);
  } catch (err) {
    handleError(err);
  }
}
```

### ⚠️ Common Mistakes
- Forgetting `return` before a callback call — code continues executing after error
- Using `async/await` inside `.forEach()` — doesn't await each iteration

### 🎯 Interview Tip
> "I always use async/await in new code. For legacy callback-based APIs, I use `util.promisify` to wrap them."

---

## Q2 — How do Promises work internally?

### ✅ Simple Explanation
A Promise is an object representing the eventual result of an async operation. It can be in one of 3 states: **pending**, **fulfilled**, or **rejected**.

### 🧠 Deep Dive
**Promise states (irreversible once settled):**
- `pending` → initial state
- `fulfilled` → resolved with a value
- `rejected` → rejected with a reason

**Promise chaining:**
- Each `.then()` returns a NEW promise
- Returning a value wraps it in `Promise.resolve(value)`
- Returning a rejected promise propagates the rejection

**Microtask queue:** Promise callbacks go into the **microtask queue**, which is drained completely before any macrotask runs. This is why Promises resolve before `setTimeout`.

### 💻 Code Example
```js
// Promise constructor pattern
function delay(ms) {
  return new Promise((resolve, reject) => {
    if (ms < 0) reject(new Error('Negative delay'));
    setTimeout(resolve, ms);
  });
}

// Promise chaining — each .then returns new Promise
fetchUser(id)
  .then(user => fetchPosts(user.id))   // returns Promise
  .then(posts => formatPosts(posts))   // returns Promise
  .catch(err => console.error(err))
  .finally(() => setLoading(false));   // always runs

// Promise combinators
const [user, posts] = await Promise.all([fetchUser(), fetchPosts()]);
const first = await Promise.race([fetchFast(), fetchSlow()]);
const results = await Promise.allSettled([p1, p2, p3]); // never rejects
const any = await Promise.any([p1, p2, p3]); // first fulfilled
```

### ⚠️ Common Mistakes
- Not returning inside `.then()` — breaks chaining (returns `undefined`)
- Forgetting `.catch()` — unhandled rejections crash Node in newer versions
- `Promise.all` fails fast — if one rejects, all reject. Use `Promise.allSettled` for independence.

### 🎯 Interview Tip
> "Promise callbacks go into the microtask queue — higher priority than setTimeout. This is key to understanding execution order."

---

## Q3 — What are the gotchas with `async/await`?

### ✅ Simple Explanation
`async/await` is syntactic sugar over Promises. An `async` function always returns a Promise. `await` pauses execution of that function until the Promise resolves.

### 🧠 Deep Dive
**Key behaviors:**
- `await` only pauses the `async` function — the event loop keeps running
- Error handling: use try/catch or `.catch()` on the awaited Promise
- `await` in loops: `for...of` with `await` runs sequentially; `map + Promise.all` runs in parallel

**Top-level await:** Available in ES modules natively. In CJS, wrap in async IIFE.

### 💻 Code Example
```js
// ❌ Sequential — slow (waits for each in order)
for (const userId of userIds) {
  const user = await fetchUser(userId); // waits before next iteration
}

// ✅ Parallel — fast
const users = await Promise.all(userIds.map(id => fetchUser(id)));

// ❌ forEach doesn't await properly
[1, 2, 3].forEach(async (id) => {
  await doSomething(id); // fire and forget, no actual sequencing
});

// ✅ Use for...of for sequential async
for (const id of [1, 2, 3]) {
  await doSomething(id);
}

// ✅ Error handling patterns
async function safeFetch(url) {
  try {
    const res = await fetch(url);
    if (!res.ok) throw new Error(`HTTP ${res.status}`);
    return await res.json();
  } catch (err) {
    console.error('Fetch failed:', err.message);
    return null; // graceful fallback
  }
}

// Parallel with error isolation
const results = await Promise.allSettled([
  fetchUser(),
  fetchPosts()
]);
const [userResult, postsResult] = results;
if (userResult.status === 'fulfilled') {
  console.log(userResult.value);
}
```

### ⚠️ Common Mistakes
- Using `async/await` inside `.map()` expecting synchronous-like behaviour — returns array of Promises
- Not handling rejected Promises — they propagate as unhandled rejections
- Awaiting inside loops when parallel execution is possible — kills performance

### 🎯 Interview Tip
> "One of the most common bugs: `array.forEach(async () => {...})` — forEach doesn't wait for async callbacks. Always use `for...of` or `Promise.all + map`."

---

## Q4 — What is `util.promisify` and when do you use it?

### ✅ Simple Explanation
`util.promisify` converts a callback-based function (error-first convention) into one that returns a Promise.

### 🧠 Deep Dive
Works with Node.js-style callbacks: `(err, result) => {}`.

Under the hood, it creates a wrapper that:
1. Calls the original function
2. In the callback, rejects if `err`, resolves with `result`

Some built-in Node modules have a `.promises` API (preferred over promisify):
- `fs.promises.readFile()`
- `dns.promises.lookup()`

### 💻 Code Example
```js
const { promisify } = require('util');
const fs = require('fs');
const dns = require('dns');

// Promisify manual
const readFile = promisify(fs.readFile);
const data = await readFile('./config.json', 'utf8');

// Better: use built-in .promises
const data2 = await fs.promises.readFile('./config.json', 'utf8');

// Custom promisify for your own callbacks
function legacyGetUser(id, callback) {
  setTimeout(() => callback(null, { id, name: 'Sandesh' }), 100);
}
const getUser = promisify(legacyGetUser);
const user = await getUser(42);

// util.promisify.custom symbol — customize behavior
legacyGetUser[util.promisify.custom] = (id) => {
  return Promise.resolve({ id, name: 'Sandesh (custom)' });
};
```

### ⚠️ Common Mistakes
- Trying to promisify functions that don't follow error-first callback convention
- Not using `fs.promises` when available (it's cleaner and official)

### 🎯 Interview Tip
> "For modern Node.js code, I use `fs.promises`, `dns.promises` etc. directly. `util.promisify` is for legacy or third-party callback APIs."

---

## Q5 — What is the difference between `Promise.all`, `Promise.allSettled`, `Promise.race`, and `Promise.any`?

### ✅ Simple Explanation
All four run Promises concurrently but differ in how they handle rejections and which result they return.

### 🧠 Deep Dive

| Method | Resolves when | Rejects when | Use case |
|---|---|---|---|
| `Promise.all` | ALL resolve | ANY rejects (fast-fail) | All required to succeed |
| `Promise.allSettled` | ALL settle (success or fail) | Never | Independent operations |
| `Promise.race` | FIRST settles (resolve or reject) | FIRST settles with rejection | Timeout pattern |
| `Promise.any` | FIRST resolves | ALL reject | First available wins |

### 💻 Code Example
```js
// Promise.all — fails fast
try {
  const [user, cart, wishlist] = await Promise.all([
    getUser(id),
    getCart(id),
    getWishlist(id)
  ]);
} catch (err) {
  // If any one fails, this catches
}

// Promise.allSettled — independent
const results = await Promise.allSettled([
  sendEmailNotification(),
  sendSMSNotification(),
  sendPushNotification()
]);
results.forEach(r => {
  if (r.status === 'rejected') console.error(r.reason);
});

// Promise.race — timeout pattern
async function fetchWithTimeout(url, ms = 5000) {
  const timeout = new Promise((_, reject) =>
    setTimeout(() => reject(new Error('Timeout')), ms)
  );
  return Promise.race([fetch(url), timeout]);
}

// Promise.any — use first available CDN
const data = await Promise.any([
  fetch('https://cdn1.example.com/data.json'),
  fetch('https://cdn2.example.com/data.json'),
  fetch('https://cdn3.example.com/data.json')
]);
```

### ⚠️ Common Mistakes
- Using `Promise.all` for independent operations — one failure kills all
- Forgetting `Promise.any` throws `AggregateError` if all reject

### 🎯 Interview Tip
> "I use `Promise.allSettled` for fire-and-forget notifications where individual failures shouldn't crash the whole flow."

---

## Q6 — What are Generator functions and how do they relate to async/await?

### ✅ Simple Explanation
Generators are functions that can be paused and resumed using `yield`. `async/await` is built on top of generators + Promises.

### 🧠 Deep Dive
- Generator: `function*` with `yield` — returns an iterator
- Each `yield` suspends the function and returns a value
- `async/await` is essentially a generator that automatically advances when a Promise resolves
- Libraries like `co` (older) implemented async flows using generators before async/await existed

### 💻 Code Example
```js
// Generator basics
function* counter() {
  yield 1;
  yield 2;
  yield 3;
}
const gen = counter();
console.log(gen.next()); // { value: 1, done: false }
console.log(gen.next()); // { value: 2, done: false }
console.log(gen.next()); // { value: 3, done: false }
console.log(gen.next()); // { value: undefined, done: true }

// Infinite sequence generator
function* idGenerator() {
  let id = 1;
  while (true) {
    yield id++;
  }
}
const ids = idGenerator();
console.log(ids.next().value); // 1
console.log(ids.next().value); // 2

// Lazy range
function* range(start, end, step = 1) {
  for (let i = start; i < end; i += step) yield i;
}
for (const n of range(0, 10, 2)) console.log(n); // 0 2 4 6 8
```

### ⚠️ Common Mistakes
- Not knowing that async/await is sugar over generators — this comes up in senior interviews
- Generators don't auto-advance — you call `.next()` manually (or use `for...of`)

### 🎯 Interview Tip
> "Generators give you lazy evaluation and coroutine-like control flow. For most async use cases, async/await is cleaner, but generators shine for infinite sequences and custom iterators."

---

## ⚡ Quick Revision Summary

- Callback → Promise → async/await is the evolution of async patterns in Node.js
- Promises have 3 states: pending, fulfilled, rejected — irreversible once settled
- `await` pauses the async function, not the event loop — microtask queue
- `async/await` in `forEach` doesn't work as expected — use `for...of` or `Promise.all(map(...))`
- `Promise.all` = fail-fast | `Promise.allSettled` = independent | `Promise.race` = timeout | `Promise.any` = first success
- Use `fs.promises` over `util.promisify(fs.readFile)` in modern Node

---

## 🏆 Top 5 Must Revise Before Interview

1. **async/await gotchas** — especially `forEach` vs `for...of`
2. **Promise.all vs allSettled** — different failure strategies
3. **Microtask queue** — why Promises resolve before setTimeout
4. **error-first callback convention** — `(err, result) => {}`
5. **Parallel vs sequential execution** — `map + Promise.all` pattern

---

## 🎤 Real Interview Questions

- *"Why doesn't `await` inside `forEach` work as expected?"*
- *"What's the difference between `Promise.all` and `Promise.allSettled`? Give a real-world use case."*
- *"How would you add a timeout to a fetch call using Promises?"*
- *"What happens if you forget to `await` a Promise?"*
