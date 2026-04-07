# 🟡 02 — Core Concepts: Async, Promises, Callbacks

---

📍 **Progress Marker**
- Last Revised: ___________
- Revision Count: 0
- Difficulty Level: 🟡 Medium

---

## Q1 — Callback Hell — what is it and how do you fix it?

### Definition

Callback Hell is a situation in JavaScript (especially in Node.js) where multiple nested callbacks make the code hard to read, maintain, and debug. It creates a “pyramid of doom” structure.

---

### Callback Pattern (Error-First)

Node.js follows an **error-first callback convention**:

```js
callback(error, result)
```

---

### Example (Callback Hell WITH Error Handling ✅)

```js
getUser(userId, function(err, user) {
  if (err) {
    console.error("Error fetching user:", err);
    return;
  }

  getOrders(user.id, function(err, orders) {
    if (err) {
      console.error("Error fetching orders:", err);
      return;
    }

    getOrderDetails(orders[0], function(err, details) {
      if (err) {
        console.error("Error fetching order details:", err);
        return;
      }

      processPayment(details, function(err, result) {
        if (err) {
          console.error("Error processing payment:", err);
          return;
        }

        console.log(result);
      });
    });
  });
});
```

---

### Problems

- Deep nesting → poor readability
    
- Difficult error handling
    
- Hard to debug and maintain
    
- Low reusability
    

---

### Solutions

#### 1. Promises

```js
getUser(userId)
  .then(user => getOrders(user.id))
  .then(orders => getOrderDetails(orders[0]))
  .then(details => processPayment(details))
  .then(result => console.log(result))
  .catch(err => console.error(err));
```

---

#### 2. Async / Await (Best Practice ⭐)

```js
async function process() {
  try {
    const user = await getUser(userId);
    const orders = await getOrders(user.id);
    const details = await getOrderDetails(orders[0]);
    const result = await processPayment(details);

    console.log(result);
  } catch (err) {
    console.error(err);
  }
}
```

---

### Key Points (Interview Ready)

- Callback Hell = deeply nested callbacks
    
- Common in async JavaScript / Node.js
    
- Error-first pattern: `callback(err, result)`
    
- Always handle errors immediately and return
    
- Solutions:
    
    - Promises
        
    - Async/Await (preferred)
        
    - Modular functions
        

---

### One-Liner

Callback Hell is deeply nested asynchronous callbacks in JavaScript that make code difficult to read and maintain, and it can be avoided using Promises, async/await, and better structuring.

---

## Q2 — How do Promises work internally?

### 🔹 What is a Promise?

> **A Promise is an object in JavaScript that is returned immediately when an asynchronous operation starts, representing a value that will be available in the future once the operation completes.**

👉 Short version:

> Promise = future result

---

### 🔹 Why Promises Were Needed

#### ❌ Before (Callbacks — messy)

```js
getUser(id, function(user) {
  getOrders(user.id, function(orders) {
    getPayment(orders[0], function(payment) {
      console.log(payment);
    });
  });
});
```

Problems:

- Nested code 😵
    
- Hard to read
    
- Error handling is messy
    

---

#### ✅ After (Promises — clean)

```js
getUser(id)
  .then(user => getOrders(user.id))
  .then(orders => getPayment(orders[0]))
  .then(payment => console.log(payment))
  .catch(err => console.error(err));
```

✔ Flat  
✔ Readable  
✔ Easy error handling

---

### 🔹 Core Idea (CRUX)

> “I’ll give you the result later — you tell me what to do when it arrives.”

---

### 🔹 Real-Life Example (Swiggy 🍔)

- You place an order → Promise created
    
- Food is being prepared → Pending
    
- Food delivered → `.then()` (success)
    
- Order cancelled → `.catch()` (error)
    

👉 You don’t go to the kitchen — you just wait for notification

---

### 🔹 How You Use a Promise

```js
getData()
  .then(data => console.log(data))   // success
  .catch(err => console.error(err))  // error
```

---

### 🔹 What Actually Happens

1. Async function starts
    
2. Returns a Promise immediately 📦
    
3. Work happens in background
    
4. When done:
    
    - `.then()` runs (success)
        
    - OR `.catch()` runs (error)
        
---

### 🔹 Who Introduced Promises?

- Concept came from **“Futures”** in computer science (1970s)
    
- Added to JavaScript in **ES6 (2015)**
    

---

### 🔹 Which Async Operations Use Promises?

#### ✅ Modern APIs (use Promises)

- API calls
    

```js
fetch(url)
```

- File system (Node.js)
    

```js
require("fs/promises").readFile()
```

- Database calls
    

```js
await User.findById(id)
```

- HTTP libraries
    

```js
axios.get("/users")
```

- Custom async
    

```js
function getResult(marks) {
  return new Promise((resolve, reject) => {
    setTimeout(() => {
      marks > 35 ? resolve("Pass") : reject("Fail");
    }, 5000);
  });
}

getResult(90)
  .then(result => console.log(result))
  .catch(err => console.log(err));
```

👉 Output after 5 sec:

```
Pass
```

---

#### ❌ Old APIs (do NOT use Promises)

- setTimeout
    

```js
setTimeout(() => {}, 1000);
```

- Old file system
    

```js
fs.readFile("file.txt", callback);
```

---

### 🔹 Key Insight

> Promises are **not automatic**  
> 👉 The API decides whether to return a Promise

---

### 🔹 Memory Trick

- `.then()` → success
    
- `.catch()` → error
    
- Promise → future value
    

---

### 🔥 Final Interview Answer

> “A Promise is an object that represents the future result of an asynchronous operation, allowing us to handle success and failure in a clean, chainable way instead of using nested callbacks.”

---

### 🚀 Ultra Short Summary

- Promise = future result
    
- Fixes callback hell
    
- Makes async code clean
    
- Used in modern APIs

---

## Q3 — What are the gotchas with `async/await`?

## 🔹 What is `async`?

> **`async` is a keyword used to declare a function that always returns a Promise.**

```js
async function getData() {
  return "Hello";
}
```

👉 Actually returns:

```js
Promise.resolve("Hello");
```

---

## 🔹 What is `await`?

> **`await` pauses the execution of an async function until a Promise is resolved or rejected.**

```js
const data = await getData();
```

👉 It unwraps the Promise and gives the actual value

---

## 🔹 Basic Example

```js
async function main() {
  const res = await fetch("https://jsonplaceholder.typicode.com/todos/1");
  const data = await res.json();
  console.log(data);
}

main();
```

---

## 🔹 Core Idea (CRUX)

> “Write async code like synchronous code.”

---

## 🔹 Under the Hood (IMPORTANT)

```js
async function getData() {
  return "Hello";
}
```

👉 Internally becomes:

```js
function getData() {
  return Promise.resolve("Hello");
}
```

---

## 🔹 Error Handling

### ✅ Using try/catch

```js
async function main() {
  try {
    const data = await getData();
  } catch (err) {
    console.error(err);
  }
}
```

---

### ❌ Without try/catch

```js
await getData(); // ❌ unhandled rejection possible
```

---

## 🔹 Where `await` Can Be Used

### 1. ✅ Inside async function

```js
async function main() {
  await getData();
}
```

---

### 2. ✅ Top-level (ONLY in ES Modules)

```js
const data = await getData();
```

👉 Requires:

- `"type": "module"` OR `.mjs`
    

---

### ❌ Not allowed

```js
function main() {
  await getData(); // ❌ SyntaxError
}
```

---

## 🔹 Sequential vs Parallel Execution

### ❌ Sequential (slow)

```js
for (const id of [1,2,3]) {
  const res = await fetch(`url/${id}`);
}
```

👉 Runs one by one

---

### ✅ Parallel (fast)

```js
await Promise.all(
  [1,2,3].map(id => fetch(`url/${id}`))
);
```

👉 Runs all together 🚀

---

## 🔹 Loop Behavior (IMPORTANT)

### ✅ Works

```js
for (const id of ids) {
  await getData(id);
}
```

---

### ❌ Does NOT work as expected

```js
ids.forEach(async (id) => {
  await getData(id); // ❌ not awaited properly
});
```

👉 `forEach` does not wait

---

## 🔹 Top-Level Await Example

```js
const res = await fetch("https://jsonplaceholder.typicode.com/todos/1");
console.log(await res.json());
```

✔ Works only in ES Modules

---

## 🔹 Common Mistakes

### 1. ❌ Forgetting `await`

```js
const data = getData(); // Promise, not actual data
```

---

### 2. ❌ Using await in non-async function

```js
function test() {
  await getData(); // ❌
}
```

---

### 3. ❌ Blocking execution unnecessarily

```js
await a();
await b();
```

👉 Better:

```js
await Promise.all([a(), b()]);
```

---

## 🔹 When to Use Async/Await vs Promise

### Use async/await when:

- You want readable, clean code
    
- Sequential logic is needed
    

### Use Promise chaining when:

- Functional chaining fits better
    
- Handling multiple async flows
    

---

## 🔹 Real-Life Analogy

- Promise → ordering food 🍔
    
- await → waiting for delivery
    

```js
const food = await orderFood();
```

👉 You pause until food arrives

---

## 🔥 Final Interview Answer

> “Async/await is syntactic sugar over Promises that allows writing asynchronous code in a synchronous style, where async functions return Promises and await pauses execution until the Promise settles.”

---

## 🚀 Ultra Short Summary

- `async` → returns Promise
    
- `await` → waits for Promise
    
- Cleaner than `.then()`
    
- Works only in async or ES modules
    
- Supports try/catch for errors

---

## Q4 — What is `util.promisify` and when do you use it?

### # Promisify in Node.js

## 📌 What is Promisify?

Promisify = Convert a **callback-based function** into a **Promise-based function**

---

## 📌 Why do we need it?

Callback-based code:

- Hard to read
    
- Callback hell
    
- Difficult error handling
    

Promise-based code:

- Cleaner
    
- Works with `async/await`
    
- Better error handling
    

---

## 📌 Built-in Promisify

Node.js provides: util.promisify

```js
const util = require('util');
const fs = require('fs');

const readFileAsync = util.promisify(fs.readFile);

const data = await readFileAsync('file.txt', 'utf8');
```

---

## 📌 How Promisify Works Internally

Callback version:

```js
fn(arg1, arg2, (err, result) => {})
```

Converted version:

```js
function promisifiedFn(...args) {
  return new Promise((resolve, reject) => {
    fn(...args, (err, result) => {
      if (err) reject(err);
      else resolve(result);
    });
  });
}
```

---

## 📌 Custom Promisify (Important)

```js
function myPromisify(fn) {
  return function (...args) {
    return new Promise((resolve, reject) => {
      fn(...args, (err, result) => {
        if (err) reject(err);
        else resolve(result);
      });
    });
  };
}
```

---

## 🔥 Interview Gold (Edge Cases)

### 1. Multiple Callback Values Lost

```js
function fn(cb) {
  cb(null, "A", "B");
}
```

```js
const result = await promisifiedFn();
console.log(result); // "A"
```

❗ Only first value is returned

✅ Fix:

```js
function myPromisifyMulti(fn) {
  return (...args) =>
    new Promise((resolve, reject) => {
      fn(...args, (err, ...results) => {
        if (err) reject(err);
        else resolve(results);
      });
    });
}
```

---

### 2. `this` Binding Issue

```js
const obj = {
  value: 10,
  getValue(cb) {
    cb(null, this.value);
  }
};
```

❌ Problem:

```js
const fn = util.promisify(obj.getValue);
await fn(); // undefined
```

✅ Fix:

```js
const fn = util.promisify(obj.getValue.bind(obj));
```

---

### 3. Only Works for Error-First Callbacks

✅ Expected:

```js
callback(err, result)
```

❌ Wrong:

```js
callback(result)
```

👉 Promisify will treat result as error

---

## 📌 When NOT to Use Promisify

- If API already supports Promises
    
- Example:
    

```js
const fs = require('fs/promises');
```

---

## 🧠 One-Line Memory Trick

Promisify assumes:  
**error-first + single result + correct this context**

---

## Q5 — What is the difference between `Promise.all`, `Promise.allSettled`, `Promise.race`, and `Promise.any`?



## ✅ 1. `Promise.all()`

### 📌 Definition

- Waits for **all promises to resolve**
    
- **Fails fast** if any promise rejects
    

### ⚙️ Syntax

```js
Promise.all(iterable)
```

### 🧠 Behavior

- ✅ Returns: Array of results
    
- ❌ Rejects: Immediately on first failure
    

### 💻 Example

```js
function fetchUser() {
  return Promise.resolve("User");
}

function fetchPosts() {
  return Promise.resolve("Posts");
}

function fetchComments() {
  return Promise.resolve("Comments");
}

Promise.all([fetchUser(), fetchPosts(), fetchComments()])
  .then(res => console.log(res)) 
  .catch(err => console.log(err));
```

### 🌍 Real-world Use Case

- Dashboard data loading (all APIs must succeed)
    

---

## ✅ 2. `Promise.allSettled()`

### 📌 Definition

- Waits for **all promises to settle (success or failure)**
    
- **Never rejects**
    

### ⚙️ Syntax

```js
Promise.allSettled(iterable)
```

### 🧠 Behavior

- Always resolves with:
    

```js
[
  { status: "fulfilled", value: ... },
  { status: "rejected", reason: ... }
]
```

### 💻 Example

```js
function api1() {
  return Promise.resolve("Success");
}

function api2() {
  return Promise.reject("Error");
}

Promise.allSettled([api1(), api2()])
  .then(res => console.log(res));
```

### 🌍 Real-world Use Case

- Show partial UI (e.g., load posts even if comments fail)
    

---

## ✅ 3. `Promise.race()`

### 📌 Definition

- Returns the **first promise that settles**
    
- Can be success OR failure
    

### ⚙️ Syntax

```js
Promise.race(iterable)
```

### 🧠 Behavior

- Fastest promise wins
    

### 💻 Example (Timeout Pattern)

```js
function fetchData() {
  return new Promise(resolve => setTimeout(() => resolve("Data"), 3000));
}

function timeout() {
  return new Promise((_, reject) => 
    setTimeout(() => reject("Timeout"), 1000)
  );
}

Promise.race([fetchData(), timeout()])
  .then(res => console.log(res))
  .catch(err => console.log(err)); // "Timeout"
```

### 🌍 Real-world Use Case

- API timeout handling
    

---

## ✅ 4. `Promise.any()`

### 📌 Definition

- Returns the **first fulfilled (successful) promise**
    
- Ignores failures unless **all fail**
    

### ⚙️ Syntax

```js
Promise.any(iterable)
```

### 🧠 Behavior

- Resolves → first success
    
- Rejects → only if all fail (`AggregateError`)
    

### 💻 Example

```js
function server1() {
  return Promise.reject("Server 1 down");
}

function server2() {
  return new Promise(resolve => setTimeout(() => resolve("Server 2 data"), 2000));
}

function server3() {
  return new Promise(resolve => setTimeout(() => resolve("Server 3 data"), 1000));
}

Promise.any([server1(), server2(), server3()])
  .then(res => console.log(res)) // Server 3 data
  .catch(err => console.log(err));
```

### 🌍 Real-world Use Case

- Multiple CDN / fallback APIs
    

---

## ⚡ Comparison Table

|Method|Waits For|Rejects When|Output|
|---|---|---|---|
|`Promise.all`|All success|Any fails ❌|Array of results|
|`Promise.allSettled`|All settled|Never ❌|Status objects|
|`Promise.race`|First settled|If first fails ❌|First result/error|
|`Promise.any`|First success|All fail ❌|First successful result|

---

## 🎯 Interview One-Liner Summary

- `Promise.all` → **All or nothing**
    
- `Promise.allSettled` → **Get all results (no failure)**
    
- `Promise.race` → **Fastest wins**
    
- `Promise.any` → **First success wins**
    

---

## 🚀 Pro Tips (Interview Gold)

- `Promise.all` is **fail-fast**
    
- `Promise.any` ignores failures (opposite of `all`)
    
- `Promise.race` is useful for **timeouts**
    
- `Promise.allSettled` is best for **resilient UI**
    

---

## 🧩 Edge Case Question

👉 What happens if all promises fail in `Promise.any`?

✔️ Answer:

```js
AggregateError: All promises were rejected
```

---

## 📌 When to Use What

- Need everything → `all`
    
- Need everything regardless → `allSettled`
    
- Need fastest → `race`
    
- Need fastest success → `any`
    

## 🔹 What are Promise Static Methods?

👉 Methods called **directly on `Promise` (not on instance)**  
👉 Used to **create, control, or wrap promises**

---

## ✅ 1. `Promise.resolve()`

### 📌 Definition

- Returns a **resolved promise**
    
- Wraps a value into a promise
    

### 💻 Example

```js
Promise.resolve("Success")
  .then(res => console.log(res)); // Success
```

### 🧠 Special Behavior

```js
Promise.resolve(Promise.resolve("Nested"))
  .then(res => console.log(res)); // Nested (auto-unwrapped)
```

### 🌍 Use Case

- Convert sync value → async
    
- Return consistent promise from function
    

---

## ❌ 2. `Promise.reject()`

### 📌 Definition

- Returns a **rejected promise**
    

### 💻 Example

```js
Promise.reject("Error occurred")
  .catch(err => console.log(err)); // Error occurred
```

### 🌍 Use Case

- Force failure in async flow
    
- Early error return
    

---

## ⚡ 3. `Promise.all()`

✔️ Already covered → All or nothing

---

## ⚡ 4. `Promise.allSettled()`

✔️ Already covered → Always returns all results

---

## ⚡ 5. `Promise.race()`

✔️ Already covered → First settled wins

---

## ⚡ 6. `Promise.any()`

✔️ Already covered → First success wins

---

## 🧠 BONUS — Instance Methods (Very Important)

👉 These are called on **promise object**

---

## 🔹 7. `.then()`

### 📌 Definition

- Handles **success**
    

```js
Promise.resolve(10)
  .then(res => res * 2)
  .then(res => console.log(res)); // 20
```

---

## 🔹 8. `.catch()`

### 📌 Definition

- Handles **errors**
    

```js
Promise.reject("Fail")
  .catch(err => console.log(err));
```

---

## 🔹 9. `.finally()`

---

## 🔹 What is `.finally()`?

👉 Runs **after promise settles (success OR failure)**  
👉 Used for **cleanup / common logic**  
👉 ❗ Does NOT receive result or error

---

## ✅ 1. Loader / Spinner Handling (Frontend)

### 📌 Problem

- Show loader while API is running
    
- Hide it regardless of success/failure
    

### 💻 Example

```js
setLoading(true);

fetchUserData()
  .then(data => setUser(data))
  .catch(err => showError(err))
  .finally(() => setLoading(false)); // ALWAYS hide loader
```

### 🌍 Real-world

👉 React / Angular apps (very common)

---

## ✅ 2. Database Connection Cleanup (Backend)

### 📌 Problem

- Open DB connection
    
- Must close it no matter what
    

### 💻 Example

```js
connectDB()
  .then(conn => queryDB(conn))
  .catch(err => console.error(err))
  .finally(() => closeDB()); // Always executed
```

### 🌍 Real-world

👉 Node.js backend services

---

## ✅ 3. File Handling / Streams

### 📌 Problem

- Open file
    
- Ensure file is closed
    

### 💻 Example

```js
openFile()
  .then(file => processFile(file))
  .catch(err => console.error(err))
  .finally(() => closeFile());
```

---

## ✅ 4. Lock / Resource Release (Concurrency)

### 📌 Problem

- Acquire lock
    
- Must release lock always
    

### 💻 Example

```js
acquireLock()
  .then(() => processCriticalTask())
  .catch(err => console.error(err))
  .finally(() => releaseLock());
```

### 🌍 Real-world

👉 Prevent race conditions in distributed systems

---

## ✅ 5. Logging / Monitoring

### 📌 Problem

- Track API execution end
    

### 💻 Example

```js
const start = Date.now();

callAPI()
  .then(() => console.log("Success"))
  .catch(() => console.log("Failed"))
  .finally(() => {
    console.log("Execution time:", Date.now() - start);
  });
```

---

## ✅ 6. Retry / Cleanup Wrapper

### 📌 Problem

- Perform cleanup after retry attempts
    

### 💻 Example

```js
retryOperation()
  .then(res => console.log(res))
  .catch(err => console.log(err))
  .finally(() => clearRetryQueue());
```

---

## 🚨 Common Mistakes

### ❌ Expecting data in finally

```js
.finally((data) => console.log(data)); // ❌ always undefined
```

---

### ❌ Writing business logic in finally

```js
.finally(() => saveUser()); // ❌ should not depend on success/failure
```

---

## ✅ Best Practices

- ✔ Use only for **cleanup / side-effects**
    
- ✔ Keep logic **independent of result**
    
- ✔ Avoid returning values (ignored)
    
- ✔ Avoid heavy logic inside
    

---

## 🎯 Interview One-Liner

👉 **"`finally()` is used for cleanup logic that must run regardless of success or failure."**

---

## 🔥 Senior-Level Insight

`.finally()` is similar to:

```js
try {
  await promise;
} catch (e) {
  // handle error
} finally {
  // cleanup
}
```

---

## 🧠 When to Use vs Not Use

|Use finally ✅|Avoid finally ❌|
|---|---|
|Stop loader|Process response|
|Close DB/file|Transform data|
|Release locks|Handle success logic|
|Logging|Handle errors|

    

---

## 🔥 Advanced (Less Known but Powerful)

---

## 🔹 10. `Promise.withResolvers()` _(Modern JS)_

### 📌 Definition

- Creates promise + exposes `resolve` & `reject`
    

```js
const { promise, resolve, reject } = Promise.withResolvers();

setTimeout(() => resolve("Done"), 1000);

promise.then(console.log);
```

### 🌍 Use Case

- Manual control (like deferred pattern)
    

---

## 🔹 11. `Promise.try()` _(Not standard everywhere)_

### 📌 Definition

- Wraps sync/async code safely
    

```js
Promise.try(() => JSON.parse("invalid"))
  .catch(err => console.log(err));
```

👉 Alternative:

```js
Promise.resolve().then(() => JSON.parse("invalid"));
```

---

## ⚡ Final Summary Table

|Type|Method|Purpose|
|---|---|---|
|Static|`Promise.resolve`|Create resolved promise|
|Static|`Promise.reject`|Create rejected promise|
|Static|`Promise.all`|All succeed or fail|
|Static|`Promise.allSettled`|Get all results|
|Static|`Promise.race`|First settled|
|Static|`Promise.any`|First success|
|Instance|`.then`|Handle success|
|Instance|`.catch`|Handle error|
|Instance|`.finally`|Always execute|

---

## 🎯 Interview One-Liners

- `Promise.resolve()` → "Wrap value into resolved promise"
    
- `Promise.reject()` → "Create rejected promise"
    
- `.then()` → "Handle success"
    
- `.catch()` → "Handle failure"
    
- `.finally()` → "Always run cleanup"
    

---

## 🧩 Trick Interview Question

👉 Is this same?

```js
return Promise.resolve(value);
```

vs

```js
return value;
```

✔️ Answer:

- Inside async function → same
    
- Outside → different (second is NOT promise)
    

---

## 🚀 Pro Tips

- Always return promises in async chains
    
- Prefer `finally` for cleanup
    
- Avoid unnecessary `new Promise()` wrapping
    
- Use `resolve` for normalization
    
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
