# 🟢 01 — Node.js Basics

---

📍 **Progress Marker**
- Last Revised: ___________
- Revision Count: 0
- Difficulty Level: 🟢 Beginner–Medium

---

## Q1 — What is Node.js and how does it work internally?

### ✅ Simple Explanation
Node.js is a JavaScript runtime built on Chrome's V8 engine.
It lets you run JavaScript on the server side.
It is single-threaded but handles concurrency via a non-blocking I/O model powered by **libuv**.
libuv provides the event loop, thread pool (default 4 threads), and async I/O.

### 🧠 Deep Dive
- V8 compiles JS to native machine code
- Node.js is NOT multi-threaded for JS execution — but it offloads I/O to the OS / thread pool
- The thread pool handles: DNS, file system, crypto, compression

**Trade-offs:**
- ✅ Excellent for I/O-bound tasks (APIs, chat apps, streaming)
- ❌ Not ideal for CPU-bound tasks (image processing, ML) — blocks the event loop

### 💻 Code Example
```js
const os = require('os');
console.log('CPUs:', os.cpus().length);
console.log('Platform:', process.platform);
console.log('Node version:', process.version);
```

### ⚠️ Common Mistakes
- Thinking Node.js is multi-threaded because it handles concurrent requests
- Confusing non-blocking I/O with parallel execution

### 🎯 Interview Tip
> "Node.js is single-threaded for JS execution but uses libuv's thread pool for blocking operations. This makes it great for I/O-heavy workloads."

### 💣 Interview Points
- Node.js handles 10,000 concurrent requests because it doesn't block the thread on I/O — it registers a callback and moves on
- The thread pool size can be changed via `UV_THREADPOOL_SIZE` env variable (max 1024)
- CPU-bound work should be offloaded using `worker_threads` or child processes

---

## Q2 — CommonJS vs ES Modules — what's the difference?

### 🔹 1. Module Systems
- **CommonJS (CJS)** → `require`, `module.exports`
- **ES Modules (ESM)** → `import`, `export`

---

### 🔹 2. Key Differences

| Feature | CommonJS (`require`) | ES Modules (`import`) |
|---|---|---|
| Loading | Synchronous | Static (analyzed before run) |
| Execution | Runtime | Compile-time |
| Usage | Anywhere | Top-level only |
| Export system | Single object | Named + Default |
| Tree-shaking | ❌ No | ✅ Yes |
| Default export | Manual | Native |
| Live bindings | ❌ No (value copy) | ✅ Yes (auto-updates) |

---

### 🔹 3. Export Patterns

#### ✅ CommonJS
```js
// Named (object)
module.exports = { add, sub };

// Default
module.exports = add;
```

#### ✅ ES Modules
```js
// Named
export const add = () => {};

// Default
export default function add() {}
```

---

### 🔹 4. Import Patterns

#### ✅ CommonJS
```js
const { add } = require('./math');
const add = require('./math');
```

#### ✅ ES Modules
```js
import add from './math.js';
import { add } from './math.js';
```

---

### 🔥 5. Most Asked Tricky Cases

#### ❗ Case 1: require + ES Module default
```js
// ES Module
export default function add() {}
```
```js
const add = require('./math'); // ❌
```
👉 Actual: `{ default: add }`

✅ Fix:
```js
const add = require('./math').default;
```

---

#### ❗ Case 2: import + CommonJS
```js
// CommonJS
module.exports = { add };
```
```js
import add from './math.js'; // ❌
```
✅ Fix:
```js
import { add } from './math.js';
```

---

#### ❗ Case 3: Default + Named
```js
export const add = () => {};
export default () => {};
```
```js
import multiply, { add } from './math.js'; // ✅
```

---

#### ❗ Case 4: Only ONE default allowed
```js
export default a;
export default b; // ❌ SyntaxError
```

---

#### ❗ Case 5: Exact name required for named exports
```js
export const add = () => {};
```
```js
import { ADD } from './math'; // ❌ — case sensitive
```

---

### 🔹 6. Dynamic Loading

#### CommonJS
```js
const lib = require('./lib'); // synchronous, anywhere
```

#### ES Modules
```js
const lib = await import('./lib.js'); // async, returns Promise
```

---

### 🧠 7. Interview One-Liners
- ES Modules are **static & optimized**
- CommonJS is **dynamic & runtime-based**
- CommonJS always exports **one object**
- `export default` → no `{}`
- `export const` → use `{}`
- Mixing both → causes `.default` bugs

---

### 🚀 8. Best Practice
- Use **ES Modules in modern apps**
- Avoid mixing CJS + ESM
- Prefer: `export default`, `export const`, `import ...`

---

### 💣 9. Real Bug Pattern
```js
const lib = require('./file');
lib.default(); // 😵 unexpected — happens when requiring an ESM file
```

---

### 🟢 Module Caching: CJS vs ESM

#### 🔥 CommonJS (CJS) — require()

**Behavior:**
- Module is executed **only once**
- Then stored in cache → `require.cache`
- Next `require()` returns **same instance**
```js
// file.js
console.log("Loaded");
module.exports = { count: 0 };
```
```js
// app.js
const a = require('./file');
const b = require('./file');

console.log(a === b); // true ✅ — same cached object
```

**Shared state (singleton):**
```js
a.count = 5;
console.log(b.count); // 5 — same reference
```

**Cache Control:**
```js
delete require.cache[require.resolve('./file')]; // force re-execution
```

---

#### 🟢 ES Modules (ESM) — import/export

**Behavior:**
- Module is also **executed only once**
- Cached internally (no `require.cache` access)
- Always returns **same instance**
```js
// file.js
console.log("Loaded");
export const obj = { count: 0 };
```
```js
// app.js
import { obj as a } from './file.js';
import { obj as b } from './file.js';

console.log(a === b); // true ✅
```

---

#### 🔥 Live Bindings — VERY IMPORTANT
```js
// file.js
export let count = 0;

export function inc() {
  count++;
}
```
```js
// app.js
import { count, inc } from './file.js';

inc();
console.log(count); // 1 ✅ — auto-updated via live binding
```

👉 ESM exports are **live references**, not copies.

---

#### ⚠️ CJS vs ESM Caching Summary

| Feature | CJS | ESM |
|---|---|---|
| Execution | Once | Once |
| Cache access | `require.cache` | Internal (no access) |
| Export type | Value copy (snapshot) | Live binding (auto-updates) |
| Cache clear | Manual (`delete require.cache`) | Not possible |

---

### 💣 Interview Points
- Both CJS & ESM execute a module only once and cache it
- CJS → runtime, mutable exports (value copy for primitives)
- ESM → compile-time, live bindings (always reflects latest value)
- Objects in CJS still share reference (reference type behavior)

---

### 🧩 Final Summary
> Use ESM for new projects.
> Use CJS only for legacy, compatibility, or dynamic loading.

---

## Q3 — What is `package.json` and what are key fields?

### 🔹 1. What is `package.json`?
> The **central configuration file** of a Node.js project.

👉 Manages: project metadata, dependencies, scripts, module system

---

### 🔹 2. Basic Example
```json
{
  "name": "my-app",
  "version": "1.0.0",
  "main": "index.js",
  "type": "module",
  "scripts": {
    "start": "node index.js"
  },
  "dependencies": {
    "express": "^4.18.2"
  }
}
```

---

### 🔥 3. Important Fields

| Field | Purpose |
|---|---|
| `name` & `version` | Project identity (required for npm publish) |
| `main` | Entry point of app |
| `type` | `"module"` = ESM, default = CJS |
| `scripts` | Run via `npm run <name>` |
| `dependencies` | Required in production |
| `devDependencies` | Development only |
| `engines` | Specifies required Node/npm version |
| `private` | Prevents accidental publish to npm |

---

### 🔹 4. package.json vs package-lock.json

| Feature | package.json | package-lock.json |
|---|---|---|
| Purpose | Declares dependencies | Locks exact versions |
| Editable | Yes | No (auto-generated) |
| Version flexibility | Uses ranges (`^`, `~`) | Exact versions |
| Created by | Developer / npm init | npm install |
| Commit to git | ✅ Yes | ✅ Yes |

---

### 🔥 5. package-lock.json
> Auto-generated file that **locks dependency versions**.

- Ensures same versions across all machines
- No "works on my machine" bugs
- Faster installs (uses cached tree)
- **Always commit to git**
- Never edit manually

---

### 🔥 6. Interview Points
- `package.json` = **declares what you want**
- `package-lock.json` = **locks what you get**

> `package.json` defines dependencies, while `package-lock.json` ensures consistent installs by locking exact versions.

---

### 🚀 Bonus Commands
```bash
npm init -y        # create package.json
npm install        # install deps
npm install x      # add dependency
npm install -D x   # add dev dependency
npm ci             # clean install (CI/CD — uses lock file strictly)
```

---

## 📦 SemVer & Version Flexibility (`^`, `~`, exact)

### 🔹 1. Semantic Versioning (SemVer)

Format: `MAJOR.MINOR.PATCH` (e.g. `4.18.2`)

| Part | Meaning |
|---|---|
| MAJOR | Breaking changes |
| MINOR | New features (backward compatible) |
| PATCH | Bug fixes |

---

### 🔹 2. Version Symbols

| Symbol | Allows | Example Range |
|---|---|---|
| `^` | Minor + Patch | `4.x.x` (not `5.x.x`) |
| `~` | Patch only | `4.18.x` |
| none | Exact only | `4.18.2` only |

#### ✅ Caret (`^`) — Most Common
```json
"express": "^4.18.2"
```
- Allows: `4.18.3`, `4.19.0`
- Blocks: `5.0.0`

#### ✅ Tilde (`~`)
```json
"express": "~4.18.2"
```
- Allows: `4.18.3`, `4.18.9`
- Blocks: `4.19.0`, `5.0.0`

#### ✅ Exact
```json
"express": "4.18.2"
```
- Only installs `4.18.2`

---

### 🔥 Special Case: Version `0.x.x`
```json
"lib": "^0.3.2"
```
- `^` becomes restrictive when MAJOR = 0
- Only allows patch updates (not minor)
- Allowed: `0.3.3`, `0.3.9`
- Blocked: `0.4.0`, `1.0.0`

> Version `0.x.x` is considered **unstable** — minor updates may be breaking.

---

### 🔹 npm install vs Version Updates

| Command | Updates Versions? | Uses Lock File? |
|---|---|---|
| `npm install` | ❌ No (usually) | ✅ Yes |
| `npm update` | ✅ Yes (within range) | ✅ Updates lock |
| `npm install x` | ✅ Yes | ✅ Updates both |
| no lock file | ✅ Yes | ❌ No |
| `npm install x@latest` | ✅ Force latest | ✅ Updates both |

> `npm install` installs from `package-lock.json`, so versions don't change unless explicitly updated or lock file is missing.

---

## Q4 — Node.js Global Objects

### 📌 What are Global Objects?
- Available everywhere (no import required)
- Provided by Node.js runtime
- Not part of JavaScript (ECMAScript)

---

### 🔥 process Object — VERY IMPORTANT

#### 1️⃣ process.env
```js
process.env.PORT       // always a string
process.env.NODE_ENV   // 'development' | 'production' | 'test'
```
- Environment variables (always strings)
- Used for configs, secrets (API keys, DB URLs)

#### 2️⃣ process.argv
```js
// node app.js hello
process.argv // → [nodePath, filePath, "hello"]
```
- `argv[0]` → node path
- `argv[1]` → file path
- `argv[2+]` → user arguments

#### 3️⃣ process.exit(code)
```js
process.exit(0)  // success
process.exit(1)  // error
```
- Stops event loop immediately

#### 4️⃣ process.cwd()
- Current working directory (where the app is **run from**)

#### 5️⃣ process.pid
- Process ID

#### 6️⃣ process.memoryUsage()
```js
// Returns: { rss, heapTotal, heapUsed, external }
```

#### 7️⃣ process.nextTick(cb)
```js
process.nextTick(() => {
  console.log('runs before any I/O callbacks');
});
```
- Runs **before** the next iteration of the event loop
- Higher priority than `setImmediate` and `setTimeout`
- See file 03 for full event loop deep dive

---

### 🔥 Important Process Events

| Event | Trigger |
|---|---|
| `process.on('exit')` | Process about to exit |
| `process.on('uncaughtException')` | Uncaught synchronous error |
| `process.on('unhandledRejection')` | Unhandled Promise rejection |
| `process.on('SIGTERM')` | OS termination signal |
| `process.on('SIGINT')` | Ctrl+C (interrupt signal) |

⚠️ `uncaughtException` → synchronous errors (`throw`)
⚠️ `unhandledRejection` → async errors (Promise without `.catch()`)

---

### 🚨 Production Best Practice
```js
process.on('uncaughtException', (err) => {
  console.error(err);
  process.exit(1);
});

process.on('unhandledRejection', (err) => {
  console.error(err);
  process.exit(1);
});
```

**Why `process.exit(1)`?**
- App is in inconsistent state
- Memory may be corrupted
- Connections may be broken
- Best approach: Crash → Restart (PM2 / Docker)

> ⚠️ Do NOT use these as primary error handling.
> Use `try/catch` for sync, `.catch()` / `async-await` for async.
> These are **last-resort safety handlers**.

---

### 🔥 SIGTERM & Graceful Shutdown

#### 📌 What is SIGTERM?
- A signal sent by the OS to **terminate a process**
- Used by: Docker, Kubernetes, `kill` command

#### 📌 Why Graceful Shutdown?
- Prevent data loss
- Close resources properly
- Avoid corrupted state

#### ✅ Full Graceful Shutdown Example
```js
const server = app.listen(3000);

process.on('SIGTERM', shutdown);
process.on('SIGINT', shutdown);

function shutdown() {
  console.log('Shutting down...');

  server.close(() => {
    console.log('Closed out remaining connections');
    // close DB, queues, etc. here
    process.exit(0);
  });

  // Force exit if graceful shutdown takes too long
  setTimeout(() => {
    console.error('Forcing shutdown');
    process.exit(1);
  }, 10000);
}
```

#### 🧠 Shutdown Flow
```
SIGTERM received
→ stop accepting new requests (server.close)
→ finish ongoing requests
→ cleanup resources (DB, queues, file handles)
→ exit process
```

#### ⚠️ What to Clean Up
- HTTP server
- Database connections
- Message queues (Kafka, RabbitMQ)
- File handles
- Background jobs / cron jobs

#### 💣 Interview Points
- `SIGTERM` → graceful (can be handled)
- `SIGKILL` → force kill (cannot be handled, no cleanup possible)
- Used in production: Docker container shutdown, Kubernetes pod termination

---

### 🔥 Other Global Objects

| Global | Description |
|---|---|
| `__dirname` | Absolute path of current file's **directory** |
| `__filename` | Absolute path of current **file** |
| `global` | Global object (like `window` in browser) |
```js
global.test = "hello";
console.log(test); // "hello"
```

⚠️ `__dirname` → file location (fixed)
⚠️ `process.cwd()` → execution location (can vary)

> ⚠️ `__dirname` and `__filename` are **not available in ESM** by default.
> Use this instead:
> ```js
> import { fileURLToPath } from 'url';
> import { dirname } from 'path';
> const __filename = fileURLToPath(import.meta.url);
> const __dirname = dirname(__filename);
> ```

---

### 🚀 process Quick Summary
```
process.env         → environment variables
process.argv        → CLI arguments
process.exit()      → stop process
process.cwd()       → where app runs from
process.pid         → process ID
process.memoryUsage() → memory stats
process.nextTick()  → run before next event loop tick

uncaughtException   → sync crash handler
unhandledRejection  → async crash handler

Both → log + exit(1)
```

---

## Q5 — What is `npm` vs `npx` vs `yarn`?

### ✅ Simple Explanation
- **npm**: Node Package Manager — installs and manages packages
- **npx**: Executes a package without installing globally
- **yarn**: Alternative package manager by Meta — faster, deterministic

### 🧠 Deep Dive

| Feature | npm | yarn (classic) | yarn berry (v2+) |
|---|---|---|---|
| Lock file | `package-lock.json` | `yarn.lock` | `yarn.lock` |
| Speed | Moderate | Fast | Very fast (PnP) |
| Workspaces | ✅ | ✅ | ✅ |
| Plug'n'Play | ❌ | ❌ | ✅ |

**npx use cases:**
- Run CLI tools without global install: `npx create-react-app`
- Run a specific version: `npx node@18 -e "console.log(process.version)"`
- Always uses latest version if not installed

### 💻 Code Example
```bash
# Install production deps only
npm install --omit=dev

# Run without installing globally
npx cowsay "Hello"

# Clean install (used in CI/CD — strict, uses lock file)
npm ci

# Check for outdated packages
npm outdated

# Audit and fix security vulnerabilities
npm audit fix
```

### ⚠️ Common Mistakes
- Running `npm install` in CI instead of `npm ci`
- Not committing `package-lock.json` to git
- Installing devDependencies in Docker production image

### 💣 Interview Points
- `npm ci` → deletes `node_modules` first, then installs from lock file exactly
- `npm ci` fails if `package-lock.json` is out of sync with `package.json`
- Always use `npm ci` in CI/CD pipelines

### 🎯 Interview Tip
> "In CI/CD pipelines, always use `npm ci` — it's deterministic, faster, and will fail if the lock file is out of sync."

---

## Q6 — `__dirname`, `__filename`, `path.join`, `require.resolve`

### 📌 Overview
- Provided by Node.js runtime (not part of JavaScript)
- Used for file paths & module resolution

---

### 🔥 __dirname
- Absolute path of current file's **directory**
```js
console.log(__dirname); // /users/sandesh/project
```

---

### 🔥 __filename
- Absolute path of current **file**
```js
console.log(__filename); // /users/sandesh/project/index.js
```

---

### 🔥 path.join()
- Safely joins multiple path segments
- Handles OS differences (`/` vs `\`)
- Removes duplicate slashes
```js
const path = require('path');
path.join(__dirname, 'folder', 'file.txt');
// → /users/sandesh/project/folder/file.txt
```

> ⚠️ Avoid string concatenation: `__dirname + "/folder/file.txt"` ❌

#### path.join vs path.resolve

| Method | Behavior |
|---|---|
| `path.join()` | Joins segments, relative to nothing |
| `path.resolve()` | Builds absolute path, processes `..` and `.` |
```js
path.resolve('/foo', 'bar', '../baz'); // → /foo/baz
path.join('/foo', 'bar', '../baz');    // → /foo/baz (same here, but behavior differs with relative paths)
```

---

### 🔥 require.resolve()
- Returns the **absolute path of a module/file without executing it**
```js
require.resolve('./index.js');
// → /users/sandesh/app/index.js

require.resolve('express');
// → /users/sandesh/app/node_modules/express/index.js
```

**Use Cases:**
1. Check if module exists (throws if not found)
2. Get actual resolved file path
3. Debug module resolution issues

> ⚠️ Does NOT import or execute the module. Only resolves the path.

**Resolution order:**
1. Local file
2. `node_modules`
3. Global paths

---

### ⚠️ Key Differences Summary

| Global | Description |
|---|---|
| `__dirname` | Current file's directory (fixed to file location) |
| `__filename` | Current file's full path |
| `process.cwd()` | Where the app was *run from* (can differ) |
| `path.join()` | Safe OS-agnostic path builder |
| `path.resolve()` | Builds absolute path, resolves `..` |
| `require.resolve()` | Get resolved module path (no execution) |

---

### 💣 Interview Points
- `__dirname` ≠ `process.cwd()` — always know which one you need
- `require.resolve` follows Node module resolution algorithm
- `path.join` is safe cross-platform; string concat is not
- `__dirname` / `__filename` are unavailable in ESM (need `import.meta.url` workaround)

---

## ⚡ Quick Revision Summary

- Node.js = V8 + libuv + event loop; single JS thread, async I/O via thread pool
- CJS: `require/module.exports` — sync, cached, value copy; ESM: `import/export` — async, live bindings, tree-shakeable
- `process.env`, `process.argv`, `process.exit()`, `process.nextTick()`, `process.on('SIGTERM')` — know all of these
- `npm ci` for CI/CD; `npx` for one-off CLI executions
- Always use `path.join(__dirname, ...)` for safe file paths
- Module caching: `require()` returns same object on repeated calls
- `__dirname` / `__filename` not available in ESM — use `import.meta.url`

---

## 🏆 Top 5 Must Revise Before Interview

1. **CJS vs ESM** — syntax, differences, live bindings, caching, interop tricky cases
2. **process.nextTick** — priority in event loop (covered deep in file 03)
3. **SIGTERM graceful shutdown** — always relevant for production/infra questions
4. **npm ci vs npm install** — shows CI/CD awareness
5. **Module caching** — `require` caches by resolved path; mutation persists

---

## 🎤 Real Interview Questions

- *"Explain how Node.js handles 10,000 concurrent requests being single-threaded."*
- *"What's the difference between `require` and `import`?"*
- *"How would you gracefully shut down a Node.js server in Kubernetes?"*
- *"What happens if you call `require()` twice for the same module?"*
- *"What's the difference between `process.nextTick` and `setImmediate`?"*
- *"Why can't you use `__dirname` in ES Modules?"*
- *"What does `npm ci` do differently from `npm install`?"*