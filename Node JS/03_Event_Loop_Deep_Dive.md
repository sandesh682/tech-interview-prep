
---
## Q1 — Explain the Node.js Event Loop in detail

### ✅ Simple Explanation

In Node.js, the **event loop** is what allows a single-threaded system to handle **thousands of concurrent operations**.

### Flow:

1. Run synchronous code
2. Push async tasks to system (libuv)
3. When done → callbacks go to queues
4. Event loop executes them phase by phase

### 🧠 Deep Dive — The 6 Phases (libuv)

![[event_loop.png]]

# Special Priority or Microtask drain (Not a phase): Microtask Queue

Microtasks are **high-priority callbacks** that execute:

> ✅ **After synchronous code**  
> ✅ **After EVERY event loop phase**  
> ❗ **Before moving to next phase**

> [!warning] Microtasks are NOT a libuv phase `process.nextTick` and Promise callbacks are **not** one of the 6 libuv phases. They are a **drain step** that runs between every phase — and between every individual callback within a phase (Timers, Check). Listing them as "Phase 1" is a common misconception.

---

## ⚡ Types of Microtasks

### 1. process.nextTick (Highest Priority)

- Runs before everything else
- Can starve event loop if overused

### 2. Promise Queue

- `.then()`, `.catch()`, `async/await`

---

## 🧠 Execution Priority

```
1. Synchronous code
2. process.nextTick queue
3. Promise queue
4. Event Loop Phases
```

---

## 🔁 Core Execution Rule

```
After EVERY phase:
→ Run nextTick queue (completely)
→ Run Promise queue (completely)
```

---

# 🧪 Experiment 1: Sync vs Microtask

## Code

```js
console.log("A");

process.nextTick(() => {
  console.log("B");
});

console.log("C");
```

## Output

```
A
C
B
```

## Insight

> ✅ Synchronous code runs first, then microtasks

---

# 🧪 Experiment 2: nextTick vs Promise

## Code

```js
Promise.resolve().then(() => {
  console.log("Promise");
});

process.nextTick(() => {
  console.log("nextTick");
});
```

## Output

```
nextTick
Promise
```

## Insight

> ✅ nextTick queue has higher priority than Promise queue

---

# 🧪 Experiment 3: Advanced Microtask Chain

## Code

```js
process.nextTick(() => console.log("NT1"));

process.nextTick(() => {
  console.log("NT2");
  process.nextTick(() => console.log("NT2-inner"));
});

process.nextTick(() => console.log("NT3"));

Promise.resolve().then(() => console.log("P1"));

Promise.resolve().then(() => {
  console.log("P2");
  process.nextTick(() => console.log("P2-inner-NT"));
});

Promise.resolve().then(() => console.log("P3"));
```

## Output

```
NT1
NT2
NT3
NT2-inner
P1
P2
P3
P2-inner-NT
```

---

## 🧠 Deep Insights

### ✅ 1. nextTick queue drains completely first

- Even dynamically added nextTick runs immediately

---

### ✅ 2. Promise queue runs only after nextTick is empty

---

### ✅ 3. Promise queue must finish completely

- Even if new nextTick is added inside Promise

---

### ✅ 4. nextTick added inside Promise runs AFTER all promises

---

## 🧠 Mental Model

```
Run sync code
→ Run nextTick (keep draining until empty)
→ Run Promise queue (finish completely)
→ If new nextTick added → run it
→ Repeat
```

---

## 💣 Important Warnings

### ❌ nextTick can block event loop

```js
process.nextTick(function repeat() {
  process.nextTick(repeat);
});
```

👉 Event loop will never proceed — nextTick queue never empties

---

## 🎯 Final Summary

- Microtasks run **between event loop phases** (not as a standalone phase)
- `process.nextTick` has **highest priority**
- Promise queue runs **after nextTick**
- Both queues are **fully drained before moving forward**

# ⏱️ Phase 1: Timers (setTimeout / setInterval)

---

## 📌 What is Timers Phase?

The **Timers phase** in Node.js executes callbacks scheduled by:

- `setTimeout`
- `setInterval`

👉 These callbacks run **only after their delay has expired** (not exactly at that time).

---

## ⚡ Execution Position

```text
[Microtask drain]
↓
Timers Phase
↓
[Microtask drain]
```

---

## 🔥 Core Rules

### 1. Microtasks have higher priority

> Even if timer delay = `0ms`, microtasks run first

---

### 2. FIFO execution (for same delay)

> Timers execute in insertion order if delay is equal

---

### 3. Delay is NOT guaranteed

```js
setTimeout(fn, 0);
```

👉 Means: _run as soon as possible_, not immediately, even though it 0, behind the scene it is considered as ~1ms

---

### 4. Microtasks interleave between timers

> After EVERY timer callback → microtasks are executed

---

# 🧪 Experiment 1: Microtask vs Timer

## Code

```js
setTimeout(() => console.log("Timer"), 0);

process.nextTick(() => console.log("nextTick"));

Promise.resolve().then(() => console.log("Promise"));
```

## Output

```text
nextTick
Promise
Timer
```

---

# 🧪 Experiment 2: Microtask inside Timer

## Code

```js
setTimeout(() => {
  console.log("T1");
  process.nextTick(() => console.log("NT1"));
}, 0);

setTimeout(() => console.log("T2"), 0);
```

## Output

```text
T1
NT1
T2
```

👉 Timer execution pauses to process microtasks before moving forward

---

# 🧪 Experiment 3: FIFO Behavior

## Code

```js
setTimeout(() => console.log("A"), 0);
setTimeout(() => console.log("B"), 0);
setTimeout(() => console.log("C"), 0);
```

## Output

```text
A
B
C
```

---

## 🧠 Mental Model

```text
Pick timer → Execute callback → Run microtasks → Next timer → Repeat
```

---

## 🎯 Summary

- Timers run after microtask drain
- `0ms` ≠ immediate execution
- Same-delay timers → FIFO
- Microtasks run after each timer callback

---

# 🌐 Phase 2: I/O Phase or Pending Callback Phase

---

## 📌 What is I/O Phase?

The **I/O (Poll) phase** in Node.js executes callbacks of completed asynchronous operations:

- File system (`fs.readFile`)
- Network requests
- Database responses

👉 This is where **real async work returns to JavaScript**

---

## ⚡ Execution Position

```text
[Microtask drain]
↓
Timers Phase
↓
[Microtask drain]
↓
I/O (Poll Phase)
↓
[Microtask drain]
```

---

## 🔥 Core Rules

### 1. Microtasks still have higher priority

> nextTick & Promise always execute before I/O callbacks

---

### 2. Runs after Timers phase

> I/O callbacks are processed only after timers (if ready)

---

### 3. Non-deterministic with 0ms timers ⚠️

> Order between I/O and `setTimeout(0)` is NOT guaranteed

Reason:

- 0ms is internally treated ~1ms
- Depends on CPU timing + event loop state

---

# 🧪 Experiment 1: Microtask vs I/O

## Code

```js
const fs = require("fs");

fs.readFile(__filename, () => {
  console.log("I/O");
});

process.nextTick(() => console.log("nextTick"));
Promise.resolve().then(() => console.log("Promise"));
```

## Output

```text
nextTick
Promise
I/O
```

---

# 🧪 Experiment 2: Timer vs I/O (Non-Deterministic)

## Code

```js
const fs = require("fs");

fs.readFile(__filename, () => {
  console.log("I/O");
});

setTimeout(() => console.log("Timer"), 0);
```

## Possible Outputs

```text
Timer
I/O
```

OR

```text
I/O
Timer
```

---

## 🧠 Insight

> Depends on whether timer is ready when event loop reaches Timers phase

---

# 🧪 Experiment 3: Forcing Timer First

## Code

```js
const fs = require("fs");

fs.readFile(__filename, () => {
  console.log("I/O");
});

// Blocking loop
for (let i = 0; i < 1e9; i++) {}

setTimeout(() => console.log("Timer"), 0);
```

## Output

```text
Timer
I/O
```

---

# 🧪 I/O vs Timer — Blocking Loop Effect

---

## 📌 Scenario

```js
const fs = require("fs");

fs.readFile(__filename, () => {
  console.log("I/O");
});

for (let i = 0; i < 1e9; i++) {} // blocking

setTimeout(() => console.log("Timer"), 0);
```

---

## ✅ Output

```text
Timer
I/O
```

---

## 🧠 Why This Happens

- `fs.readFile` runs in background (async)
- Blocking loop delays event loop
- During this delay → timer expires (0ms → ~1ms internally)

👉 When event loop starts:

- **Timers phase** → Timer is ready → runs first
- **I/O phase** → runs after

---

## ⚡ Key Insight

```text
Blocking delay → ensures timer is ready → Timer runs before I/O
```

---

## ⚖️ Comparison

### Without blocking

```text
Timer / I/O (not guaranteed)
```

### With blocking

```text
Timer → I/O (guaranteed)
```

---

## 🎯 Summary

- `setTimeout(0)` needs time to expire
- Blocking loop gives that time
- Event loop executes whichever is ready first

---

## 🧠 Mental Model

```text
If timers ready → process timers first
Else → process I/O callbacks
After each → run microtasks
```

---

## 🎯 Summary

- I/O phase handles async results (fs, network)
- Runs after Timers phase (if timers ready)
- Microtasks always execute before I/O
- `setTimeout(0)` vs I/O → order not guaranteed
# ⚡ Phase 3: Idle, Prepare Phase (Internal)
- Used internally by Node.js
- Not something you interact with directly
# ⚡ Phase 4: Poll Phase (Most Important)

---

## ⚡ Position in Event Loop

```text
[Microtask drain]
↓
Timers
↓
[Microtask drain]
↓
I/O (Poll)
↓
[Microtask drain]
↓
Check (setImmediate)
↓
[Microtask drain]
```

---

## 🔥 Core Rules

### 1. Runs after I/O phase

> Check queue executes once I/O phase completes (or is skipped)

---

### 2. I/O uses polling (important ⚠️)

> I/O callbacks are added **only after operation finishes**

- `fs.readFile` does NOT immediately go to queue
- Event loop **polls** to check completion

---

### 3. Empty I/O queue case (🔥 key insight)

> If I/O is NOT ready when poll phase runs:

👉 Event loop **skips I/O queue**  
👉 Moves to **Check phase**

---

# 🧪 Experiment: setImmediate vs I/O

## Code

```js
const fs = require("fs");

fs.readFile(__filename, () => {
  console.log("I/O");
});

setImmediate(() => {
  console.log("Immediate");
});
```

---

## ✅ Output

```text
Immediate
I/O
```

---

## 🧠 Step-by-Step Explanation

### 1️⃣ fs.readFile starts

- Goes to background (not ready yet)

---

### 2️⃣ Event loop → I/O phase

- Checks I/O queue
- ❌ Empty (file not read yet)

---

### 3️⃣ Moves to Check phase

- Executes:

```text
Immediate
```

---

### 4️⃣ Next loop iteration

- Now I/O is ready
- Executes:

```text
I/O
```

---

# 💣 Key Insight

```text
I/O callbacks are NOT queued until operation finishes
```

👉 If event loop already passed I/O phase:

- Callback waits for **next iteration**

---

# ⚖️ Comparison

### I/O ready early

```text
I/O → Immediate
```

### I/O NOT ready (common case)

```text
Immediate → I/O
```

---

## 🧠 Mental Model

```text
Check I/O queue
→ If empty → go to Check phase
→ Run setImmediate
→ Next cycle → process I/O
```

---

## 🎯 Summary

- `setImmediate` runs in Check phase
- Check phase comes after I/O
- I/O uses polling → may not be ready
- If I/O not ready → Check phase executes first
- I/O callback may shift to next loop cycle

---

# ⚡ Phase 5 : Check Phase

---

## 📌 Role of Check Phase

Handles callbacks scheduled via:

```js
setImmediate()
```

👉 Executes **after I/O (Poll) phase** in Node.js

---

## ⚡ Execution Priority (Full Context)

```text
Microtask drain (nextTick → Promise)
↓
Timers
↓
I/O (Poll)
↓
Check (setImmediate)
```

---

## 🔥 Core Rules

### 1. Lowest priority among async queues (before close)

> Runs after microtask drain, timers, and I/O

---

### 2. Microtasks can interrupt Check phase

> After EACH `setImmediate` callback → microtasks run

---

### 3. I/O must be completed or skipped before Check

> Check phase only runs after I/O polling step

---

# 🧪 Experiment 1: setImmediate inside I/O

## Code

```js
const fs = require("fs");

fs.readFile(__filename, () => {
  console.log("I/O");

  setImmediate(() => {
    console.log("Immediate");
  });
});
```

## Output

```text
I/O
Immediate
```

---

## 🧠 Insight

- I/O completes → callback runs
- Then Check phase executes `setImmediate`

---

# 🧪 Experiment 2: Microtasks inside I/O

## Code

```js
const fs = require("fs");

fs.readFile(__filename, () => {
  console.log("I/O");

  process.nextTick(() => console.log("nextTick"));
  Promise.resolve().then(() => console.log("Promise"));

  setImmediate(() => console.log("Immediate"));
});
```

## Output

```text
I/O
nextTick
Promise
Immediate
```

---

## 🧠 Insight

> Microtasks inside I/O callback run BEFORE Check phase

---

# 🧪 Experiment 3: Multiple setImmediate

## Code

```js
setImmediate(() => {
  console.log("A");

  Promise.resolve().then(() => console.log("Microtask"));
});

setImmediate(() => console.log("B"));
```

## Output

```text
A
Microtask
B
```

---

## 🧠 Insight

> Microtasks execute between individual `setImmediate` callbacks

---

# 🧪 Experiment 4: Timer vs setImmediate ⚠️

## Code

```js
setTimeout(() => console.log("Timer"), 0);
setImmediate(() => console.log("Immediate"));
```

## Output (non-deterministic)

```text
Timer
Immediate
```

OR

```text
Immediate
Timer
```

---

## 🧠 Insight

```text
Order depends on timing + event loop state
```

- Outside I/O → ❌ not guaranteed
- Inside I/O → ✅ `setImmediate` runs first

> [!info] Why setImmediate wins inside I/O When you are already executing inside an I/O callback, the Timers phase has **already been passed** for this loop iteration. So `setTimeout(0)` cannot fire until the **next iteration**. `setImmediate` (Check phase) is still ahead in the current iteration — that is why it always wins when called from inside I/O.

---

## 💣 Important Warning

### ❌ Recursive setImmediate can starve I/O

```js
setImmediate(function repeat() {
  setImmediate(repeat);
});
```

👉 Each `setImmediate` schedules the next one in the following loop iteration's Check phase. I/O callbacks that were not yet ready keep getting deferred because the Check phase stays busy across iterations.

> [!warning] Unlike `process.nextTick`, recursive `setImmediate` does not block microtasks — but it **does** continuously defer I/O callbacks that are not yet in the queue.

---

## 🧠 Mental Model

```text
After I/O:
→ Run setImmediate (one by one)
→ After each → run microtasks
→ Continue
```

---

## 🎯 Summary

- `setImmediate` runs in Check phase (libuv Phase 5)
- Executes after I/O (if I/O present)
- Microtasks always interrupt Check execution
- Multiple `setImmediate` → interleaved with microtasks
- `setTimeout(0)` vs `setImmediate` → not guaranteed (outside I/O)
- Inside I/O → `setImmediate` always wins because Timers phase is already past

# 🔚 Phase 6: Close Queue

---

## 📌 What is Close Phase?

The **Close phase** in Node.js executes callbacks related to **resource cleanup**, such as:

- `socket.on('close')`
- stream close events

👉 This is the **last phase** in one event loop iteration

---

## ⚡ Position in Event Loop

```text
[Microtask drain]
↓
Timers
↓
[Microtask drain]
↓
I/O (Poll)
↓
[Microtask drain]
↓
Check (setImmediate)
↓
[Microtask drain]
↓
Close (final phase)
↓
[Microtask drain]
```

---

## 🔥 Core Rules

### 1. Always executes last

> Close callbacks run after Timers, I/O, and Check phases

---

### 2. Used for cleanup

> Triggered when resources are closed (sockets, streams)

---

### 3. Microtasks still interrupt

> After each close callback → microtasks are executed

---

# 🧪 Experiment: Close Event

## Code

```js
const net = require("net");

const server = net.createServer((socket) => {
  socket.on("close", () => {
    console.log("Close");
  });

  socket.destroy();
});

server.listen(3000, () => {
  const client = net.createConnection(3000);
});
```

---

## ✅ Output

```text
Close
```

---

## 🧠 Insight

- Socket is destroyed → triggers `close` event
- Callback goes to **Close queue**
- Executes after all other phases

---

# ⚡ Full Event Loop — Correct Execution Order

> [!important] Read this carefully The table below shows the **actual execution sequence** per loop iteration. Microtasks (nextTick + Promise) are **not a phase** — they are a drain step that runs after every phase and after every individual callback.

```text
── Loop iteration start ──────────────────────────

  [Sync code runs to completion]

  [Microtask drain]
    → nextTick queue (drain completely)
    → Promise queue (drain completely)

  [Phase 1 — Timers]
    → Execute each expired setTimeout / setInterval callback
    → After EACH callback → microtask drain

  [Phase 2 — I/O / Poll]
    → Execute ready I/O callbacks
    → After EACH callback → microtask drain
    → If queue empty → move on (don't wait)

  [Phase 3 — Check]
    → Execute each setImmediate callback
    → After EACH callback → microtask drain

  [Phase 4 — Close]
    → Execute close event callbacks
    → After EACH callback → microtask drain

── Loop iteration end → repeat or exit ───────────
```

---

## 💣 Important Behavior

### Microtasks run between everything

```text
After each phase
AND
After each callback (Timers / Check)
```

---

## 🧠 Mental Model

```text
Run all phases
→ Finally run Close callbacks
→ Then next loop iteration starts
```

---

## 🎯 Summary

- Close queue handles cleanup callbacks
- Always runs last in event loop cycle
- Triggered by resource closing events
- Microtasks still have higher priority
- Microtasks are a **drain step**, not a phase — they run after every phase and every callback
---

## Q2 — `process.nextTick` vs `setImmediate` vs `setTimeout(fn, 0)`


| API                 | Runs in         | Runs when                                               |
| ------------------- | --------------- | ------------------------------------------------------- |
| `process.nextTick`  | Microtask drain | Before ANY event loop phase, after current operation    |
| `Promise.then`      | Microtask drain | After nextTick queue is empty                           |
| `setTimeout(fn, 0)` | Timers phase    | After ~1ms delay, in next (or current) loop iteration   |
| `setImmediate`      | Check phase     | After I/O poll phase, in current or next loop iteration |

---

## 1️⃣ process.nextTick

### What it is

`process.nextTick` is **not part of the event loop phases** (libuv). It is a Node.js-specific mechanism that queues a callback into the **nextTick queue**, which is drained:

- After the current synchronous operation completes
- After every single event loop phase
- After every individual callback inside Timers and Check phases

### Use case

> Use when you want something to run **after the current call stack clears, but before any I/O or timers**.

Typical pattern — deferring an error to allow caller to attach a handler:

```js
function readData(callback) {
  if (!data) {
    return process.nextTick(() => callback(new Error("No data")));
    // Without nextTick, callback fires before caller can attach it
  }
  callback(null, data);
}
```

### 💣 Danger — Can starve the event loop

```js
process.nextTick(function repeat() {
  process.nextTick(repeat); // ❌ nextTick queue never empties
});
// I/O, timers, setImmediate — NONE of these will ever run
```

> [!warning] Recursive `process.nextTick` completely blocks the event loop. The queue must fully drain before any phase runs, so infinite nextTick = infinite starvation.

---

## 2️⃣ setTimeout(fn, 0)

### What it is

Schedules a callback in the **Timers phase** of the event loop. The `0` delay is converted internally to `~1ms` minimum by libuv. The callback runs only when:

1. The Timers phase is reached
2. The delay has expired

### Key behaviour

- Delay is **minimum**, not exact — OS scheduling and event loop state both affect it
- Multiple `setTimeout(fn, 0)` calls execute in **FIFO order** within the same Timers phase
- Microtasks drain **after each individual timer callback** (not just after all timers)

### Use case

> Use when you need a **rough deferral** and timing precision does not matter. Not recommended when you specifically want "after I/O" — use `setImmediate` instead.

```js
console.log("start");
setTimeout(() => console.log("timeout"), 0);
console.log("end");

// Output:
// start
// end
// timeout
```

---

## 3️⃣ setImmediate

### What it is

Schedules a callback in the **Check phase** (libuv Phase 5), which runs after the I/O (Poll) phase. Despite the name, it is **not immediate** — it waits for the current loop iteration to reach the Check phase.

### Key behaviour

- Always runs **after I/O callbacks** in the same loop iteration (if I/O was present)
- If I/O queue was empty, Check phase still runs (after Poll phase is skipped)
- Microtasks drain after **each individual** `setImmediate` callback

### Use case

> Use when you want to execute **after I/O in the current loop iteration**, or when you want to break up CPU work without delaying I/O callbacks.

```js
const fs = require("fs");

fs.readFile(__filename, () => {
  setImmediate(() => console.log("runs right after this I/O callback"));
});
```

### 💣 Danger — Can defer I/O callbacks across iterations

```js
setImmediate(function repeat() {
  setImmediate(repeat);
});
// I/O callbacks that were not yet ready keep getting pushed to the next iteration
```

> [!warning] Unlike `process.nextTick`, recursive `setImmediate` does **not** starve microtasks — but it continuously defers I/O callbacks that arrive between iterations.

---

# 🧪 Experiments

---

## 🧪 Experiment 1: Basic Priority Order

### Code

```js
console.log("1: sync");

setTimeout(() => console.log("4: setTimeout"), 0);

setImmediate(() => console.log("5: setImmediate"));

Promise.resolve().then(() => console.log("3: Promise"));

process.nextTick(() => console.log("2: nextTick"));

console.log("1: sync end");
```

### Output

```
1: sync
1: sync end
2: nextTick
3: Promise
4: setTimeout
5: setImmediate
```

### Insight

> ✅ nextTick → Promise → Timers → Check. Sync code always runs first, then microtask drain, then event loop phases in order.

---

## 🧪 Experiment 2: setTimeout(0) vs setImmediate — Outside I/O (Non-Deterministic)

### Code

```js
setTimeout(() => console.log("setTimeout"), 0);
setImmediate(() => console.log("setImmediate"));
```

### Possible Outputs

```
setTimeout
setImmediate
```

OR

```
setImmediate
setTimeout
```

### Insight

> ⚠️ When called from the **main module** (outside I/O), the order is **not guaranteed**.
> 
> Reason: When the event loop starts its first iteration, it checks the Timers phase first. Whether the `~1ms` timer has expired by then depends on CPU load and OS scheduling. If it has → Timer runs first. If it hasn't → Check phase runs first (setImmediate).

> [!warning] Never rely on the ordering of `setTimeout(0)` vs `setImmediate` outside of an I/O callback.

---

## 🧪 Experiment 3: setTimeout(0) vs setImmediate — Inside I/O (Deterministic)

### Code

```js
const fs = require("fs");

fs.readFile(__filename, () => {
  setTimeout(() => console.log("setTimeout"), 0);
  setImmediate(() => console.log("setImmediate"));
});
```

### Output (always)

```
setImmediate
setTimeout
```

### Insight

> ✅ Inside an I/O callback, `setImmediate` **always** wins.

> [!info] Why setImmediate always wins inside I/O When you are executing inside an I/O callback, you are already in the I/O (Poll) phase. The Timers phase has **already been passed** for this loop iteration — `setTimeout(0)` cannot fire until the **next iteration**. The Check phase (`setImmediate`) is still ahead in the **current iteration**, so it always runs first.

---

## 🧪 Experiment 4: nextTick vs setImmediate vs setTimeout — All Together Inside I/O

### Code

```js
const fs = require("fs");

fs.readFile(__filename, () => {
  console.log("I/O");

  setTimeout(() => console.log("setTimeout"), 0);
  setImmediate(() => console.log("setImmediate"));
  Promise.resolve().then(() => console.log("Promise"));
  process.nextTick(() => console.log("nextTick"));
});
```

### Output (always)

```
I/O
nextTick
Promise
setImmediate
setTimeout
```

### Insight

> ✅ Even inside I/O, microtasks (nextTick → Promise) drain before any further phases run. ✅ Then Check phase (setImmediate) runs — it's ahead of Timers in the current iteration. ✅ setTimeout fires in the **next** loop iteration's Timers phase.

---

## 🧪 Experiment 5: Nested nextTick vs nested setImmediate

### Code

```js
process.nextTick(() => {
  console.log("NT1");
  process.nextTick(() => console.log("NT2 — nested"));
});

setImmediate(() => {
  console.log("SI1");
  setImmediate(() => console.log("SI2 — nested"));
});

setTimeout(() => {
  console.log("ST1");
  setTimeout(() => console.log("ST2 — nested"), 0);
}, 0);
```

### Output

```
NT1
NT2 — nested
ST1
SI1
ST2 — nested
SI2 — nested
```

### Insight

> ✅ Nested `nextTick` runs immediately (same drain cycle) — NT2 runs before any phase. ✅ Nested `setTimeout(0)` fires in the **next** Timers phase (next loop iteration). ✅ Nested `setImmediate` fires in the **next** Check phase (next loop iteration).

> [!info] This is the clearest demonstration of the drain-vs-phase difference:
> 
> - `nextTick` drains recursively in the same microtask drain
> - `setImmediate` and `setTimeout` each defer their nested call by a full loop iteration

---

# 🎯 When to Use Which

|Situation|Use|
|---|---|
|Run after current call stack, before I/O|`process.nextTick`|
|Emit errors asynchronously in custom APIs|`process.nextTick`|
|Run after I/O in the same loop iteration|`setImmediate`|
|Break up long CPU work without blocking I/O|`setImmediate`|
|Rough time-based deferral|`setTimeout(fn, delay)`|
|"After current tick" in browser-compatible code|`Promise.resolve().then(...)`|

> [!tip] Node.js docs themselves recommend preferring `setImmediate` over `process.nextTick` in most cases — it is safer because it cannot starve the event loop.

---

# 💣 Common Interview Traps

---

### Trap 1 — "setTimeout(0) runs immediately"

❌ False. It has a minimum delay of ~1ms and must wait for the Timers phase. `process.nextTick` is the closest thing to "run immediately after this".

---

### Trap 2 — "setImmediate runs before setTimeout(0)"

❌ Not always. Only guaranteed **inside an I/O callback**. Outside I/O, the order is non-deterministic.

---

### Trap 3 — "nextTick is part of the event loop"

❌ False. `process.nextTick` is not a libuv phase. It is a Node.js-level drain step that intercepts between phases and between individual callbacks.

---

### Trap 4 — "setImmediate is immediate"

❌ Misleading name. It runs in the Check phase — after I/O polling. It is "immediate relative to I/O", not "immediate relative to now".

---

### Trap 5 — "Recursive setImmediate is safe"

⚠️ Partially true. It does not block microtasks. But it continuously defers I/O callbacks that arrive between iterations, which can cause latency issues in I/O-heavy servers.

---

## Q3 — What is libuv and what is the thread pool?

## 🧠 Definition

- libuv = C library that powers Node.js async behavior
    
- Provides:
    
    - Event Loop
        
    - Async I/O
        
    - Thread Pool
        
- Enables non-blocking execution in single-threaded JS
    

---

## 🎯 Core Idea

Node.js is single-threaded, but libuv offloads work to keep it scalable.

---

## ⚙️ Execution Model

JS Code  
↓  
libuv  
↓  
Decision:

- OS Async (Networking)
    
- Thread Pool (FS, DNS, Crypto)  
    ↓  
    Event Loop  
    ↓  
    Callback Execution
    

---

## ⚖️ Two Strategies

### OS-level Async (Preferred)

- Uses: epoll (Linux), kqueue (macOS), IOCP (Windows)
    
- Used for: HTTP, TCP, sockets
    
- No threads involved
    
- Highly scalable
    

### Thread Pool (Fallback)

- Default size = 4
    
- Used for:
    
    - File System (fs)
        
    - DNS (dns.lookup)
        
    - Crypto (pbkdf2, bcrypt)
        
- Simulates async behavior using threads
    

---

## 📊 Operations Mapping

- HTTP → OS Async
    
- File System → Thread Pool
    
- DNS → Thread Pool
    
- Crypto → Thread Pool
    

---

## 🔄 Event Loop (High-level)

Timers → I/O → Poll → Check → Close

Executes callbacks after async work completes

---

## 🧠 Key Insight

OS Async > Thread Pool

Thread pool is fallback, not primary mechanism

---

## 🚫 Why not OS async for everything?

- File systems are blocking
    
- DNS uses blocking system calls
    
- Crypto is CPU-bound
    
- Cross-platform inconsistencies
    

---

## ⚠️ Interview Traps

- “All async uses thread pool” → Wrong
    
- “Node.js has no threads” → Wrong
    
- “libuv = only event loop” → Wrong
    
- “Async always means OS-level non-blocking” → Wrong
    

---

## 🚀 One-line Summary

libuv enables Node.js non-blocking behavior using event loop + OS async + thread pool

---
## Q4 — What is thread pool in node.js?

## 🧠 Definition

- Thread pool = set of worker threads managed by libuv
    
- Used to execute blocking or CPU-heavy tasks in background
    
- Keeps event loop non-blocking
    
- Default size = 4
    

---

## 🎯 Core Idea

Thread pool provides **concurrency across multiple tasks**, not parallelism for a single task

---

## ⚙️ How It Works

JS Code  
↓  
libuv  
↓  
Task Queue (FIFO)  
↓  
Thread Pool (4 threads)  
↓  
Execution  
↓  
Event Loop  
↓  
Callback Execution

---

## 📊 Execution Model

- Each task is assigned to **one thread only**
    
- Tasks are **NOT split across multiple threads**
    

Example:

1 task → 1 thread  
4 tasks → 4 threads  
10 tasks → 4 threads + 6 queued

---

## 🔄 Visual Model

Thread Pool (4 threads)

Thread 1 → Task  
Thread 2 → Task  
Thread 3 → Task  
Thread 4 → Task

Remaining tasks wait in queue

---

## 📦 What Uses Thread Pool

- File System (fs)
    
- Crypto (pbkdf2, bcrypt)
    
- DNS (dns.lookup)
    
- Compression (zlib)
    

---

## ❌ What Does NOT Use Thread Pool

- HTTP / Network calls
    
- Database calls
    
- setTimeout / setImmediate
    

(Handled by OS async)

---

## ⚠️ Critical Behaviors

### Limited Parallelism

- Only N threads run in parallel (default 4)
    
- बाकी tasks wait in queue
    

---

### FIFO Queue

- Tasks executed in order of arrival
    
- No priority system
    

---

### Blocking per Thread

- One slow task blocks one thread
    
- Reduces available capacity
    

---

### Global Resource

- Shared across entire Node.js process
    
- All modules compete for same threads
    

---

## 🔥 Important Insight

Thread pool improves **throughput**, not **latency of a single task**

- 1 heavy task → uses only 1 thread
    
- Remaining threads stay idle
    

---

## ⚠️ Real-world Problem

100 bcrypt requests:

- 4 processed
    
- 96 waiting
    
- Increased response time
    

This is called **thread pool saturation**

---

## ⚙️ Configuration

UV_THREADPOOL_SIZE=8 node app.js

---

## 🧠 When to Increase

- Heavy file operations
    
- Crypto-heavy workloads
    
- Compression tasks
    

---

## ❌ Why Not Too Many Threads

- CPU contention
    
- Context switching overhead
    
- Memory usage increase
    
- Performance degradation
    

---

## 🔄 Context Switching

- CPU switches between threads
    
- Each switch adds overhead
    
- Too many threads → slower system
    

---

## ⚖️ Thread Pool vs Worker Threads

Thread Pool:

- For I/O + small CPU tasks
    
- Limited scaling
    

Worker Threads:

- For heavy CPU tasks
    
- True parallelism
    

---

## ⚠️ Interview Traps

- “All async uses thread pool” → Wrong
    
- “Increasing threads always improves performance” → Wrong
    
- “Thread pool splits one task across threads” → Wrong
    
- “Each request gets its own thread” → Wrong
    

---

## 🚀 One-line Summary

Thread pool executes blocking tasks in background using limited threads while keeping event loop non-blocking

---

## 🎯 Interview Answer

Node.js uses libuv’s thread pool (default size 4) to handle blocking operations like file system, DNS, and crypto tasks, ensuring the event loop remains non-blocking.

---
## Q6 — How are different asynchronous callbacks like timers, I/O, promises, setImmediate, and close events scheduled and executed in Node.js?

## 🧠 Core Idea

- Event loop executes **callbacks**, not the actual work
    
- Work may be done by:
    
    - OS async (network)
        
    - Thread pool (fs, dns, crypto)
        
    - Or internally (timers, promises)
        

---

## ⚙️ Event Loop Flow (Simplified)

Microtasks (Promises)  
↓  
Timers (setTimeout / setInterval)  
↓  
I/O Callbacks  
↓  
Check Phase (setImmediate)  
↓  
Close Callbacks

---

## 🔮 Promises (Microtasks)

- Handled by **V8 microtask queue**
    
- Runs:
    
    - After current execution
        
    - Before next event loop phase
        

👉 Highest priority

---

## ⏱️ Timers (setTimeout / setInterval)

- Registered in **libuv timer queue**
    
- Uses system clock (no thread, no OS async I/O)
    
- When time expires → callback queued in **timers phase**
    

---

## 📥 I/O Callbacks Phase

- Executes callbacks of completed I/O
    

### Two sources:

1. OS Async (network)
    
    - epoll / kqueue / IOCP
        
    - No thread pool
        
2. Thread Pool (fs, dns, crypto)
    
    - Work done in background threads
        

👉 Important:  
I/O phase runs callbacks, not the work itself

---

## ⚡ setImmediate (Check Phase)

- Scheduled in **check phase**
    
- Runs after I/O phase
    

---

## ❌ Close Callbacks

- Runs when resources are closed
    

Example:

- socket.on('close', ...)
    

---

## 🔄 How Callbacks Reach Event Loop

### OS Async (Network)

JS → libuv → OS → Event Loop → I/O Phase → Callback

---

### Thread Pool (fs, crypto)

JS → libuv → Thread Pool → Event Loop → I/O Phase → Callback

---

### Timers

JS → libuv (timer queue) → Timer expires → Timers Phase

---

### Promises

JS → V8 microtask queue → Executed immediately after current code

---

### setImmediate

JS → libuv → Check Phase

---

### Close

Resource closes → Close Phase

---

## ⚠️ Key Insights

- Event loop executes **callbacks only**
    
- Not all async operations use thread pool
    
- Promises run before timers
    
- I/O phase depends on source (OS or thread pool)
    

---

## ⚠️ Interview Traps

- “All async goes through thread pool” → Wrong
    
- “setTimeout uses threads” → Wrong
    
- “Promises use libuv” → Wrong (handled by V8)
    
- “I/O phase means thread pool” → Wrong
    

---

## 🚀 One-line Summary

Event loop schedules and executes callbacks from multiple sources: microtasks (promises), timers, I/O (OS or thread pool), setImmediate, and close events.

---

## 🎯 Interview Answer

Promises are handled by the microtask queue (V8), timers by libuv’s timer queue, I/O callbacks come from either OS async or thread pool, setImmediate runs in the check phase, and close callbacks run when resources are cleaned up.

---
## Q5 — What blocks the event loop and how do you fix it?


Event loop is **blocked when synchronous code runs too long**, preventing:

- I/O callbacks
    
- timers
    
- promises  
    from executing on time.
    

👉 Node.js is **single-threaded (main thread)** → if blocked = app becomes unresponsive.

---

## 🔴 1. CPU-Intensive / Heavy Computation

### ❌ Problem

```js
function fibonacci(n) {
  if (n <= 1) return n;
  return fibonacci(n - 1) + fibonacci(n - 2);
}
fibonacci(45); // blocks event loop for seconds
```

### ✅ Fix 1 — Worker Threads (true parallelism)

```js
const { Worker, isMainThread, parentPort, workerData } = require('worker_threads');

if (isMainThread) {
  const worker = new Worker(__filename, { workerData: { n: 45 } });

  worker.on('message', result => console.log('Result:', result));
  worker.on('error', err => console.error(err));
} else {
  function fibonacci(n) {
    if (n <= 1) return n;
    return fibonacci(n - 1) + fibonacci(n - 2);
  }

  parentPort.postMessage(fibonacci(workerData.n));
}
```

### ✅ Fix 2 — Chunking (yield control)

```js
function fibChunked(n, cb) {
  function step(a, b, count) {
    if (count === 0) return cb(a);
    setImmediate(() => step(b, a + b, count - 1));
  }
  step(0, 1, n);
}
```

---

## 🔴 2. Synchronous File System Calls

### ❌ Problem

```js
const fs = require('fs');

const data = fs.readFileSync('/large-file.csv', 'utf8'); // blocks
processData(data);
```

### ✅ Fix

```js
fs.readFile('/large-file.csv', 'utf8', (err, data) => {
  if (err) throw err;
  processData(data);
});
```

### ✅ Async/Await

```js
const { readFile } = require('fs/promises');

const data = await readFile('/large-file.csv', 'utf8');
processData(data);
```

---

## 🔴 3. Synchronous Crypto / Hashing

### ❌ Problem

```js
const crypto = require('crypto');

const hash = crypto.pbkdf2Sync(password, salt, 100000, 64, 'sha512'); // blocks
```

### ✅ Fix (libuv thread pool)

```js
crypto.pbkdf2(password, salt, 100000, 64, 'sha512', (err, hash) => {
  console.log(hash.toString('hex'));
});
```

### ✅ Promisified

```js
const { promisify } = require('util');
const pbkdf2 = promisify(crypto.pbkdf2);

const hash = await pbkdf2(password, salt, 100000, 64, 'sha512');
```

---

## 🔴 4. Large JSON Parsing

### ❌ Problem

```js
const data = JSON.parse(hugeJsonString); // blocks (sync)
```

### ✅ Fixes

- Stream parsing
    

```js
const { parser } = require('stream-json');
fs.createReadStream('huge.json').pipe(parser());
```

- Worker threads
    
- Paginate data at DB level
    

---

## 🔴 5. process.nextTick / Promise Starvation

### ❌ Problem (starvation)

```js
function recursiveNextTick() {
  process.nextTick(recursiveNextTick);
}
recursiveNextTick(); // blocks I/O forever
```

### ✅ Fix

```js
function recursiveImmediate() {
  setImmediate(recursiveImmediate);
}
```

👉 `process.nextTick` runs **before event loop phases**  
👉 `setImmediate` allows loop to continue (poll phase)

---

## 🔴 6. DNS + Thread Pool Saturation

### ⚠️ Problem

- `dns.lookup` uses **libuv thread pool**
    
- Default pool size = **4**
    
- Heavy usage → queue → delays
    

### ✅ Fix 1 — Use c-ares (non-thread pool)

```js
const dns = require('dns').promises;

const addresses = await dns.resolve4('example.com');
```

### ✅ Fix 2 — Increase thread pool

```bash
UV_THREADPOOL_SIZE=16 node server.js
```

---

## 🔴 7. Regex Catastrophic Backtracking (ReDoS)

### ❌ Problem

```js
const re = /^(a+)+$/;
re.test('aaaaaaaaaaaaaaaaaaaab'); // exponential backtracking
```

### ✅ Fixes

- Rewrite regex
    
- Use `safe-regex`
    
- Timeout / AbortSignal (Node 22+)
    
- Run in worker thread
    

---

## 🧵 libuv Thread Pool — Key Facts

|Property|Value|
|---|---|
|Default size|4|
|Max size|1024|
|Set via|UV_THREADPOOL_SIZE|
|Used by|fs, crypto, dns.lookup, zlib|
|NOT used by|network I/O (epoll/kqueue)|

> ⚠️ Gotcha:  
> If 4 threads are busy → next tasks WAIT (queue), even if event loop is free.

---

## 📊 Monitoring Event Loop Lag

### ✅ Manual Detection

```js
let lastCheck = Date.now();

setInterval(() => {
  const now = Date.now();
  const lag = now - lastCheck - 1000;

  if (lag > 100) {
    console.warn(`Event loop lag: ${lag}ms`);
  }

  lastCheck = now;
}, 1000);
```

### ✅ perf_hooks (Best)

```js
const { monitorEventLoopDelay } = require('perf_hooks');

const h = monitorEventLoopDelay({ resolution: 20 });
h.enable();

setInterval(() => {
  console.log('p50 lag:', h.percentile(50) / 1e6, 'ms');
  console.log('p99 lag:', h.percentile(99) / 1e6, 'ms');
  h.reset();
}, 5000);
```

### ✅ Production Tools

- clinic.js → `clinic doctor -- node server.js`
    
- 0x → flamegraphs
    
- Node flag → `--prof`
    

---

## 🧠 Quick Interview Summary

👉 Event loop blocks when:

- CPU-heavy sync code
    
- Sync APIs (fs, crypto)
    
- Large JSON parsing
    
- nextTick recursion
    
- Thread pool saturation
    
- Bad regex (ReDoS)
    

👉 Solutions:

- Worker Threads (CPU tasks)
    
- Async APIs (fs, crypto)
    
- Chunking (setImmediate)
    
- Streaming (large data)
    
- Increase thread pool
    
- Avoid nextTick abuse
    

---

## ⚡ Golden Rule

> ❌ NEVER block the event loop  
> ✅ Always keep it **free to process callbacks**

---
## Q7 — Why might Node.js not be suitable for CPU-intensive tasks?

👉 Node.js runs on a **single-threaded event loop**

- One main thread handles:
    
    - incoming requests
        
    - callbacks
        
    - timers
        
    - promises
        

❗ If this thread is busy → **everything stops**

---

## 🔴 Problem: CPU-Intensive Tasks

CPU-heavy operations:

- large loops
    
- recursion (e.g., fibonacci)
    
- image/video processing
    
- data crunching
    

👉 These run **synchronously on main thread**

---

## ⚠️ What Goes Wrong?

### 1. Event Loop Blocking

```js
function fibonacci(n) {
  if (n <= 1) return n;
  return fibonacci(n - 1) + fibonacci(n - 2);
}

app.get('/heavy', (req, res) => {
  const result = fibonacci(45); // ❌ blocks event loop
  res.send(result);
});
```

🚫 While this runs:

- No other request is processed
    
- No I/O callbacks executed
    
- App becomes **unresponsive**
    

---

### 2. No Parallelism by Default

- Node.js uses **only one CPU core**
    
- Other cores remain unused
    

👉 Heavy task = full load on single core

---

### 3. Poor Scalability

👉 One slow request can block thousands:

- 1 CPU-heavy request  
    ❌ blocks  
    → 1000 fast API requests
    

---

### 4. Thread Pool Misconception

👉 libuv thread pool is **NOT for your JS logic**

Used only for:

- fs (file system)
    
- crypto
    
- dns.lookup
    
- zlib
    

❗ Your CPU-heavy JS code still runs on **main thread**

---

## 📉 Result

- High latency
    
- Low throughput
    
- Bad user experience
    
- Server appears "frozen"
    

---

## ✅ When Node.js is GOOD

Best for:

- I/O-heavy apps
    
- APIs
    
- real-time systems (chat, websockets)
    
- streaming
    

👉 Because it handles **non-blocking I/O efficiently**

---

## ✅ Solutions for CPU Tasks

### ✔️ 1. Worker Threads (Best)

```js
const { Worker } = require('worker_threads');
```

- True parallelism
    
- Offload heavy computation
    

---

### ✔️ 2. Break Work into Chunks

```js
setImmediate(() => { /* next chunk */ });
```

- Prevent long blocking
    

---

### ✔️ 3. Microservices Architecture

- Move heavy work to:
    
    - Python
        
    - Go
        
    - Java
        

---

### ✔️ 4. Background Jobs / Queues

- BullMQ
    
- RabbitMQ
    
- Kafka
    

---

### ✔️ 5. Clustering

```bash
node cluster.js
```

- Use multiple processes (multi-core)
    
- Still not per-request parallelism
    

---

## 🎯 Interview Answer (Perfect)

> Node.js is not suitable for CPU-intensive tasks because it uses a single-threaded event loop. Heavy computations block the event loop, preventing it from handling other requests, which leads to poor performance and unresponsiveness. Since it doesn’t utilize multiple CPU cores by default, CPU-heavy tasks should be offloaded to worker threads or external services.

---

## ⚡ Golden Rule

> ❌ Never block the event loop  
> ✅ Keep it free for handling requests

---
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
