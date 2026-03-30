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
libuv provides the event loop, thread pool (default 4 threads), and async I/O

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

---

## Q2 — CommonJS vs ES Modules — what's the difference?


# 🔹 1. Module Systems

- **CommonJS (CJS)** → `require`, `module.exports`
    
- **ES Modules (ESM)** → `import`, `export`

---

# 🔹 2. Key Differences

|Feature|CommonJS (`require`)|ES Modules (`import`)|
|---|---|---|
|Loading|Synchronous|Static (analyzed before run)|
|Execution|Runtime|Compile-time|
|Usage|Anywhere|Top-level only|
|Export system|Single object|Named + Default|
|Tree-shaking|❌ No|✅ Yes|
|Default export|Manual|Native|

---

# 🔹 3. Export Patterns

## ✅ CommonJS

```js
// Named (object)
module.exports = { add, sub };

// Default
module.exports = add;
```

## ✅ ES Modules

```js
// Named
export const add = () => {};

// Default
export default function add() {}
```

---

# 🔹 4. Import Patterns

## ✅ CommonJS

```js
const { add } = require('./math');
const add = require('./math');
```

## ✅ ES Modules

```js
import add from './math.js';
import { add } from './math.js';
```

---

# 🔥 5. MOST ASKED TRICKY CASES

## ❗ Case 1: require + ES Module default

```js
// ES Module
export default function add() {}
```

```js
const add = require('./math'); // ❌
```

👉 Actual:

```js
{ default: add }
```

✅ Fix:

```js
const add = require('./math').default;
```

---

## ❗ Case 2: import + CommonJS

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

## ❗ Case 3: Default + Named

```js
export const add = () => {};
export default () => {};
```

```js
import multiply, { add } from './math.js'; // ✅
```

---

## ❗ Case 4: Only ONE default

```js
export default a;
export default b; // ❌
```

---

## ❗ Case 5: Exact name required

```js
export const add = () => {};
```

```js
import { ADD } from './math'; // ❌
```

---

# 🔹 6. Dynamic Loading

## CommonJS

```js
const lib = require('./lib');
```

## ES Modules

```js
const lib = await import('./lib.js');
```

---

# 🧠 7. Interview One-Liners

- ES Modules are **static & optimized**
    
- CommonJS is **dynamic & runtime-based**
    
- CommonJS always exports **one object**
    
- `export default` → no `{}`
    
- `export const` → use `{}`
    
- Mixing both → causes `.default` bugs
    

---

# 🚀 8. Best Practice

- Use **ES Modules in modern apps**
    
- Avoid mixing CJS + ESM
    
- Prefer:
    

```js
export default
export const
import ...
```

---

# 💣 9. Real Bug Pattern

```js
const lib = require('./file');

lib.default(); // 😵 unexpected
```

---

# 🟢 Module Caching: CJS vs ESM

---

## 🔥 CommonJS (CJS) – require()

### 📌 Behavior

- Module is executed **only once**
    
- Then stored in cache → `require.cache`
    
- Next `require()` returns **same instance**
    

---

### ✅ Example

```js
// file.js
console.log("Loaded");
module.exports = { count: 0 };
```

```js
// app.js
const a = require('./file');
const b = require('./file');

console.log(a === b); // true
```

---

### 🧠 Key Points

- Runs only once
    
- Cached after first load
    
- Shared state (singleton)
    

```js
a.count = 5;
console.log(b.count); // 5
```

---

### ⚠️ Cache Control

```js
delete require.cache[require.resolve('./file')];
```

---

# 🟢 ES Modules (ESM) – import/export

---

## 📌 Behavior

- Module is also **executed only once**
    
- Cached internally (no `require.cache`)
    
- Always returns **same instance**
    

---

### ✅ Example

```js
// file.js
console.log("Loaded");
export const obj = { count: 0 };
```

```js
// app.js
import { obj as a } from './file.js';
import { obj as b } from './file.js';

console.log(a === b); // true
```

---

## 🧠 Key Points

- Singleton behavior (same instance)
    
- Cached after first evaluation
    
- Static imports (top-level only)
    

---

## 🔥 Important Difference

### Live Bindings (VERY IMPORTANT 💣)

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
console.log(count); // 1 ✅ (auto-updated)
```

👉 ESM exports are **live references**

---

## ⚠️ CJS vs ESM Difference

```text
CJS → value copy (snapshot-like behavior), however as objcets are refernce type, it works for objects
ESM → live binding (auto updates), if variable is updated in one file, it gets updated where it is imported.
```

---

## ⚠️ No Manual Cache Access

- No equivalent of `require.cache`
    
- Cannot easily force reload
    

---

# 💣 Interview Points

- Both CJS & ESM:
    
    - Execute module only once
        
    - Cache result
        
- Differences:
    
    - CJS → runtime, mutable exports
        
    - ESM → compile-time, live bindings
        

---

# 🚀 Quick Summary

CJS:

- require()
    
- cached in require.cache
    
- same instance returned
    
- can clear cache manually
    

ESM:

- import/export
    
- cached internally
    
- same instance returned
    
- live bindings (auto updates)
    
- no manual cache control

# 🧩 Final Summary

> Use ESM for new projects.  
> Use CJS only for legacy or compatibility or dynamic loading

---

## Q3 — What is `package.json` and what are key fields?

---

# 🔹 1. What is `package.json`?

> The **central configuration file** of a Node.js project.

👉 Manages:

- Project metadata
    
- Dependencies
    
- Scripts
    
- Module system
    

---

# 🔹 2. Basic Example

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

# 🔥 3. Important Fields

## ✅ `name` & `version`

- Project identity (required for npm publish)
    

## ✅ `main`

- Entry point of app
    

## ✅ `type`

- `"module"` → ES Modules (`import`)
    
- default → CommonJS (`require`)
    

## ✅ `scripts`

```bash
npm run start
```

## ✅ `dependencies`

- Required in production
    

## ✅ `devDependencies`

- Only for development
    

---

# 🔹 4. What does it do?

- Tracks installed packages
    
- Enables `npm install`
    
- Defines how app runs
    
- Controls module system
    

---

# 🔹 5. package.json vs package-lock.json

|Feature|package.json|package-lock.json|
|---|---|---|
|Purpose|Declares dependencies|Locks exact versions|
|Editable|Yes|No (auto-generated)|
|Version flexibility|Uses ranges (`^`, `~`)|Exact versions|
|Created by|Developer / npm init|npm install|

---

# 🔥 6. What is `package-lock.json`?

> Auto-generated file that **locks dependency versions**.

---

## ✅ Why it exists?

👉 Ensures:

- Same versions across all machines
    
- No “works on my machine” bugs
    
- Faster installs (uses cached tree)
    

---

## 🔍 Example

```json
"express": "^4.18.2"
```

👉 In `package.json` → flexible

```json
"express": "4.18.2"
```

👉 In `package-lock.json` → exact

---

## ⚠️ Important Rules

- Always **commit `package-lock.json`**
    
- Never edit manually
    
- Auto-updated on `npm install`
    

---

# 🔥 7. Interview Points (High Value)

- `package.json` = **declares what you want**
    
- `package-lock.json` = **locks what you get**
    

---

# 🧠 One-Liner

> `package.json` defines dependencies, while `package-lock.json` ensures consistent installs by locking exact versions.

---

# 🚀 Bonus Commands

```bash
npm init -y      # create package.json
npm install      # install deps
npm install x    # add dependency
npm install -D x # add dev dependency
```

---

# 🧩 Final Summary

> package.json = project blueprint  
> package-lock.json = exact dependency snapshot

---

# 📦 Node.js Version Flexibility (`^`, `~`, exact) — Interview Notes

---

# 🔹 1. Semantic Versioning (SemVer)

Format:

```text
MAJOR.MINOR.PATCH
```

Example:

```text
4.18.2
```

- **MAJOR** → Breaking changes
    
- **MINOR** → New features (backward compatible)
    
- **PATCH** → Bug fixes
    

---

# 🔹 2. Version Symbols in `package.json`

## ✅ Caret (`^`) — Most Common

```json
"express": "^4.18.2"
```

👉 Allows:

- Minor + Patch updates (same MAJOR)
    

### ✅ Allowed:

- 4.18.3
    
- 4.19.0
    

### ❌ Not Allowed:

- 5.0.0
    

💡 Rule:

> `^` = flexible within same MAJOR

---

## ✅ Tilde (`~`)

```json
"express": "~4.18.2"
```

👉 Allows:

- Patch updates only (same MINOR)
    

### ✅ Allowed:

- 4.18.3
    
- 4.18.9
    

### ❌ Not Allowed:

- 4.19.0
    
- 5.0.0
    

💡 Rule:

> `~` = flexible within same MINOR

---

## ✅ Exact Version

```json
"express": "4.18.2"
```

👉 Allows:

- Only exact version
    

---

# 🔹 3. Comparison Table

|Symbol|Allows Changes|Example Range|
|---|---|---|
|`^`|Minor + Patch|4.x.x (not 5.x.x)|
|`~`|Patch only|4.18.x|
|none|Exact|4.18.2 only|

---

# 🔹 4. Real-world Behavior

```json
"express": "^4.18.2"
```

👉 Different machines may install:

- 4.18.2
    
- 4.19.1
    

✅ Controlled by **package-lock.json**

---

# 🔹 5. Role of `package-lock.json`

- Locks **exact versions**
    
- Ensures same install across machines
    
- Prevents unexpected updates
    

💡 Key Idea:

> `package.json` = version range  
> `package-lock.json` = exact version

---

# 🔥 6. Special Case: Version `0.x.x`

👉 When MAJOR = 0, behavior changes

```json
"lib": "^0.3.2"
```

### ❗ Behavior:

- `^` becomes restrictive
    
- Only allows patch updates within same minor
    

### ✅ Allowed:

- 0.3.3
    
- 0.3.9
    

### ❌ Not Allowed:

- 0.4.0
    
- 1.0.0
    

---

## 💡 Why?

> Version `0.x.x` is considered **unstable**, so minor updates may break code.

---

## 🔁 Comparison

|Symbol|Normal (>=1.0.0)|Version 0 Behavior|
|---|---|---|
|`^`|Minor + Patch|Patch only within same minor|
|`~`|Patch only|Patch only (same)|

---

# 🧠 7. Interview One-Liners

- `^` → minor + patch updates
    
- `~` → patch updates only
    
- exact → strict version
    
- `package-lock.json` ensures consistency
    
- Version `0` → `^` behaves like `~`
    

---

# 🚀 8. Best Practice

- Use `^` in most projects
    
- Rely on `package-lock.json` for stability
    
- Use exact versions for critical systems
    

---

# 🧩 Final Summary

> `^` = flexible (minor + patch)  
> `~` = safer (patch only)  
> exact = strict  
> version 0 = more restrictive

---
# 🔹 npm install vs Version Updates (IMPORTANT)

## ❗ Does `npm install` update versions?

👉 **No (by default)**

- Installs from **`package-lock.json`**
    
- Ensures same versions across all machines
    

---

## ✅ When versions DO update

### 1. No `package-lock.json`

```bash
npm install
```

👉 Installs latest version within allowed range (`^`, `~`)

---

### 2. Installing a new package

```bash
npm install lodash
```

👉 Installs latest version  
👉 Updates both:

- `package.json`
    
- `package-lock.json`
    

---

### 3. Using update command

```bash
npm update
```

👉 Updates packages within allowed range

---

### 4. Force latest version

```bash
npm install express@latest
```

👉 Ignores range → installs newest version

---

## 🔥 Summary Table

|Command|Updates Versions?|Uses Lock File?|
|---|---|---|
|`npm install`|❌ No (usually)|✅ Yes|
|`npm update`|✅ Yes|✅ Updates lock|
|`npm install x`|✅ Yes|✅ Updates both|
|no lock file|✅ Yes|❌ No|

---

## 🧠 Interview One-Liner

> `npm install` installs from `package-lock.json`, so versions don’t change unless explicitly updated or lock file is missing.

---

## Q4 — Node.js: Global Objects

## 📌 What are Global Objects?

- Available everywhere (no import required)
    
- Provided by Node.js runtime
    
- Not part of JavaScript (ECMAScript)
    

---

# 🔥 process Object (VERY IMPORTANT)

## 📌 Definition

- Global object representing the currently running Node.js process
    

---

## 🔥 Must-Know Properties & Methods

### 1️⃣ process.env

- Environment variables (ALWAYS strings)
    

process.env.PORT  
process.env.NODE_ENV

Used for:

- Configs
    
- Secrets (API keys, DB URLs)
    

---

### 2️⃣ process.argv

- Command-line arguments
    

node app.js hello

process.argv → [nodePath, filePath, "hello"]

Key Points:

- argv[0] → node path
    
- argv[1] → file path
    
- argv[2+] → user arguments
    

---

### 3️⃣ process.exit(code)

- Terminates the process
    

process.exit(0) → exit after success (e.g. task is completed) 
process.exit(1) → error (uncaughtException)

Important:

- Stops event loop immediately
    

---

### 4️⃣ process.cwd()

- Current working directory (where app is run)
    

---

### 5️⃣ process.pid

- Process ID
    

---

### 6️⃣ process.memoryUsage()

- Memory stats
    

Returns:

- rss
    
- heapTotal
    
- heapUsed
    
- external
    

---

## 🔥 Important Events (Interview Gold)

### process.on('exit')

- Runs when process is about to exit
    

### process.on('uncaughtException')

- Handles uncaught synchronous errors
    

### process.on('unhandledRejection')

- Handles unhandled Promise rejections (async errors)

# SIGTERM & Graceful Shutdown (Node.js)

---

## 📌 What is SIGTERM?

- A signal sent by the OS to **terminate a process**
    
- Default signal used by:
    
    - Docker
        
    - Kubernetes
        
    - `kill` command
        

---

## 📌 Why Graceful Shutdown?

- Prevent data loss
    
- Close resources properly
    
- Avoid corrupted state
    

---

# 🔥 Basic Handling

```js
process.on('SIGTERM', () => {
  console.log('SIGTERM received');
  process.exit(0);
});
```

---

# ⚠️ Problem with above

- Immediate exit ❌
    
- Does NOT:
    
    - close DB
        
    - stop server
        
    - finish requests
        

---

# ✅ Graceful Shutdown (Correct Way)

```js
const server = app.listen(3000);

process.on('SIGTERM', () => {
  console.log('SIGTERM received');

  server.close(() => {
    console.log('HTTP server closed');

    // close DB, queues, etc.
    process.exit(0);
  });
});
```

---

# 🔥 What to Clean Up

- HTTP server (stop accepting new requests)
    
- Database connections
    
- Message queues (Kafka, RabbitMQ)
    
- File handles
    
- Background jobs
    

---

# 🧠 Flow

SIGTERM received  
→ stop new requests  
→ finish ongoing requests  
→ cleanup resources  
→ exit process

---

# ⚠️ Important Notes

- `server.close()`:
    
    - Stops new connections
        
    - Allows existing requests to finish
        
- Always add timeout fallback:
    

```js
setTimeout(() => {
  console.error('Force shutdown');
  process.exit(1);
}, 10000);
```

---

# 💣 Interview Points

- SIGTERM ≠ SIGKILL
    
    - SIGTERM → graceful (can handle)
        
    - SIGKILL → force kill (cannot handle)
        
- Used in production environments:
    
    - Docker container shutdown
        
    - Kubernetes pod termination
        

---

# 🚀 Full Example

```js
const server = app.listen(3000);

process.on('SIGTERM', shutdown);
process.on('SIGINT', shutdown);

function shutdown() {
  console.log('Shutting down...');

  server.close(() => {
    console.log('Closed out remaining connections');
    process.exit(0);
  });

  setTimeout(() => {
    console.error('Forcing shutdown');
    process.exit(1);
  }, 10000);
}
```

---

# 🧠 Quick Summary

SIGTERM → termination signal

Graceful shutdown:

- stop new requests
    
- finish ongoing work
    
- clean resources
    
- exit safely
    

SIGKILL → cannot be handled

Always:  
handle SIGTERM + cleanup + timeout fallback
    

---

## ⚠️ Difference

uncaughtException → synchronous errors (throw)  
unhandledRejection → async errors (Promise without catch)

---

## 🚨 Production Best Practice

process.on('uncaughtException', (err) => {  
console.error(err);  
process.exit(1);  
});

process.on('unhandledRejection', (err) => {  
console.error(err);  
process.exit(1);  
});

---

## 💣 Why process.exit(1)?

- App is in inconsistent state
    
- Memory may be corrupted
    
- Connections may be broken
    

Best approach:  
Crash → Restart (PM2 / Docker)

---

## 🧠 Golden Rules

- Do NOT use these as primary error handling
    
- Use try/catch for sync
    
- Use .catch() / async-await for async
    
- These are last-resort safety handlers
    

---

# 🔥 Other Global Objects

## __dirname

- Absolute path of current file’s directory
    

## __filename

- Absolute path of current file
    

## global

- Global object in Node.js (like window in browser)
    

global.test = "hello"  
console.log(test)

---

## ⚠️ Important Difference

__dirname → file location  
process.cwd() → execution location

---

# 🚀 Final Quick Summary

process → current running app info & control

process.env → environment variables  
process.argv → CLI arguments  
process.exit() → stop process  
process.cwd() → where app runs  
process.pid → process ID  
process.memoryUsage() → memory stats

uncaughtException → sync crash  
unhandledRejection → async crash

Both → log + exit(1)

__dirname → file directory  
__filename → file path  
global → global object

---

## Q5 — What is `npm` vs `npx` vs `yarn`?

### ✅ Simple Explanation
- **npm**: Node Package Manager — installs packages
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

### 💻 Code Example
```bash
# Install production deps only
npm install --omit=dev

# Run without installing
npx cowsay "Hello"

# Clean install (used in CI/CD)
npm ci

# Check for outdated packages
npm outdated

# Audit security vulnerabilities
npm audit fix
```

### ⚠️ Common Mistakes
- Running `npm install` in CI instead of `npm ci` (npm ci is stricter and faster)
- Not committing `package-lock.json` to git
- Installing dev dependencies in Docker production image

### 🎯 Interview Tip
> "In CI/CD pipelines, always use `npm ci` — it's deterministic, faster, and will fail if lock file is out of sync."

---

## Q6 — What is __dirname, __filename, path.join, require.resolve?

---

## 📌 Overview

- Provided by Node.js runtime (not part of JavaScript)
    
- Used for file paths & module resolution
    

---

# 🔥 __dirname

## 📌 Definition

- Absolute path of current file’s directory
    

console.log(__dirname)

### 🧠 Key Point

- Represents file location (NOT execution location)
    

---

# 🔥 __filename

## 📌 Definition

- Absolute path of current file
    

console.log(__filename)

---

# 🔥 path.join()

## 📌 Definition

- Safely joins multiple path segments
    
- From built-in `path` module
    

const path = require('path')  
path.join(__dirname, 'folder', 'file.txt')

---

## ✅ Why use it?

- Handles OS differences (`/` vs `\`)
    
- Removes duplicate slashes
    
- Creates clean paths
    

---

## ⚠️ Avoid this

__dirname + "/folder/file.txt"

---

# 🔥 require.resolve()

## 📌 Definition

- Returns the **absolute path of a module/file without executing it**
    

require.resolve('./file')

---

## ✅ Example

const path = require('path')

console.log(require.resolve('./index.js'))

// Output:  
// /users/sandesh/app/index.js

---

## 🔥 Use Cases

### 1️⃣ Check if module exists

require.resolve('express')

---

### 2️⃣ Get actual file path

const filePath = require.resolve('./config.json')

---

### 3️⃣ Debug module resolution

- Helps understand where Node is loading module from
    

---

## ⚠️ Important

- Does NOT import the module
    
- Only resolves path
    
- Throws error if module not found
    

---

# ⚠️ Key Differences

__dirname → current file directory  
__filename → current file path  
path.join() → safely build paths  
require.resolve() → get resolved module path

---

# 💣 Interview Points

- __dirname ≠ process.cwd()
    
- __dirname → file location
    
- cwd() → execution location
    
- require.resolve follows Node module resolution:
    
    1. Local file
        
    2. node_modules
        
    3. Global paths
        

---

# 🚀 Quick Summary

__dirname → folder path  
__filename → file path  
path.join() → safe path builder  
require.resolve() → find exact file path (no execution)

---

## ⚡ Quick Revision Summary

- Node.js = V8 + libuv + event loop; single JS thread, async I/O
- CJS: `require/module.exports` — sync, cached; ESM: `import/export` — async, live bindings, tree-shakeable
- `process.env`, `process.argv`, `process.exit()`, `process.on('SIGTERM')` — know all of these
- `npm ci` for CI/CD; `npx` for one-off CLI executions
- Always use `path.join(__dirname, ...)` for safe file paths
- Module caching: `require()` returns same object on repeated calls

---

## 🏆 Top 5 Must Revise Before Interview

1. **CJS vs ESM** — syntax, differences, when to use which
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
