# 🟢 01 — Node.js Basics

---

📍 **Progress Marker**
- Last Revised: ___________
- Revision Count: 0
- Difficulty Level: 🟢 Beginner–Medium

---

## Q1 — What is Node.js and how does it work internally?

### ✅ Simple Explanation
Node.js is a JavaScript runtime built on Chrome's V8 engine. It lets you run JavaScript on the server side. It is single-threaded but handles concurrency via a non-blocking I/O model powered by **libuv**.

### 🧠 Deep Dive
- V8 compiles JS to native machine code
- libuv provides the event loop, thread pool (default 4 threads), and async I/O
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

### ✅ Simple Explanation
- **CommonJS (CJS)**: `require()` / `module.exports` — synchronous, older standard
- **ES Modules (ESM)**: `import` / `export` — async, modern standard, tree-shakeable

### 🧠 Deep Dive
| Feature        | CommonJS          | ES Modules                              |
| -------------- | ----------------- | --------------------------------------- |
| Syntax         | `require()`       | `import`                                |
| Loading        | Synchronous       | Asynchronous                            |
| Live binding   | No                | Yes (exports are live)                  |
| File extension | `.js` (default)   | `.mjs` or `"type":"module"`             |
| `__dirname`    | ✅ Available       | ❌ Not available (use `import.meta.url`) |
| Tree shaking   | ❌ No              | ✅ Yes                                   |
| Circular deps  | Handled (partial) | Handled (live bindings)                 |

**Module caching**: Both systems cache modules after first load. `require()` returns the same instance on repeated calls.

### 💻 Code Example
```js
// CommonJS
const express = require('express');
module.exports = { handler };

// ES Module
import express from 'express';
export const handler = () => {};

// ESM __dirname equivalent
import { fileURLToPath } from 'url';
import { dirname } from 'path';
const __filename = fileURLToPath(import.meta.url);
const __dirname = dirname(__filename);
```

### ⚠️ Common Mistakes
- Mixing `require` and `import` in the same file (breaks in Node without babel)
- Forgetting that ESM files in Node need `"type": "module"` in `package.json` or `.mjs` extension
- Not knowing that `module.exports` is cached — mutating it doesn't re-run the file

### 🎯 Interview Tip
> "I prefer ESM for new projects due to tree-shaking and live bindings. CJS is still dominant in existing Node ecosystems."

---

## Q3 — What is `package.json` and what are key fields?

### ✅ Simple Explanation
`package.json` is the manifest file for a Node project — it defines metadata, dependencies, scripts, and configuration.

### 🧠 Deep Dive
**Critical fields:**
- `main`: entry point for CJS
- `module`: entry point for ESM (used by bundlers)
- `exports`: modern way to define multiple entry points
- `dependencies` vs `devDependencies`: production vs dev-only
- `peerDependencies`: expected to be installed by consumer
- `engines`: specifies Node.js version compatibility

**Semantic versioning (semver):**
- `^1.2.3` — allows minor + patch updates (1.x.x)
- `~1.2.3` — allows only patch updates (1.2.x)
- `1.2.3` — exact version

### 💻 Code Example
```json
{
  "name": "mern-server",
  "version": "1.0.0",
  "type": "module",
  "main": "index.js",
  "scripts": {
    "start": "node index.js",
    "dev": "nodemon index.js",
    "test": "jest"
  },
  "dependencies": {
    "express": "^4.18.0",
    "mongoose": "^7.0.0"
  },
  "devDependencies": {
    "nodemon": "^3.0.0",
    "jest": "^29.0.0"
  },
  "engines": {
    "node": ">=18.0.0"
  }
}
```

### ⚠️ Common Mistakes
- Putting test libraries in `dependencies` instead of `devDependencies`
- Not locking versions in production (`package-lock.json` or `yarn.lock` must be committed)

### 🎯 Interview Tip
> "The `exports` field gives fine-grained control over what consumers can import — important for building libraries."

---

## Q4 — What is the `process` object in Node.js?

### ✅ Simple Explanation
`process` is a global object that provides information and control over the current Node.js process.

### 🧠 Deep Dive
Key properties and methods:
- `process.env` — environment variables
- `process.argv` — CLI arguments array (index 0 = node, 1 = script)
- `process.exit(code)` — terminate process (0 = success, 1 = error)
- `process.nextTick(cb)` — queue callback before I/O events (see Event Loop file)
- `process.cwd()` — current working directory
- `process.memoryUsage()` — heap stats
- `process.uptime()` — seconds since process started

### 💻 Code Example
```js
// Read env variable with fallback
const PORT = process.env.PORT || 3000;

// Parse CLI args
// node app.js --env production
const args = process.argv.slice(2);
console.log(args); // ['--env', 'production']

// Graceful shutdown
process.on('SIGTERM', () => {
  console.log('SIGTERM received. Closing server...');
  server.close(() => {
    process.exit(0);
  });
});

// Uncaught exception handler (last resort)
process.on('uncaughtException', (err) => {
  console.error('Uncaught Exception:', err);
  process.exit(1); // Always exit after uncaughtException
});
```

### ⚠️ Common Mistakes
- Not handling `uncaughtException` — process crashes silently in some setups
- Using `process.exit()` without closing DB connections first (causes data loss)
- Accessing `process.env` without dotenv loaded in local dev

### 🎯 Interview Tip
> "Always set up a SIGTERM handler for graceful shutdown in production — important for Kubernetes deployments."

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

## Q6 — What is `__dirname`, `__filename`, and `require.resolve`?

### ✅ Simple Explanation
- `__dirname` — absolute path of the current file's directory
- `__filename` — absolute path of the current file
- `require.resolve()` — returns the resolved path of a module without loading it

### 🧠 Deep Dive
These are CommonJS-specific globals. In ESM, you must reconstruct them:

```js
// ESM equivalent
import { fileURLToPath } from 'url';
import { dirname, join } from 'path';

const __filename = fileURLToPath(import.meta.url);
const __dirname = dirname(__filename);
```

`require.resolve` is useful for checking if a module exists before requiring it.

### 💻 Code Example
```js
// CJS — build paths relative to current file
const path = require('path');
const configPath = path.join(__dirname, '..', 'config', 'db.js');

// Check if module exists
try {
  require.resolve('some-optional-package');
  const pkg = require('some-optional-package');
} catch {
  console.log('Optional package not installed');
}
```

### ⚠️ Common Mistakes
- Using relative paths like `./config/db.js` instead of `path.join(__dirname, ...)` — breaks when script is run from a different working directory

### 🎯 Interview Tip
> "Always use `path.join(__dirname, ...)` for file paths in Node — never hardcode paths with `/` or rely on `process.cwd()`."

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
