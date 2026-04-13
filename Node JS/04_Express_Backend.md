# 🔵 04 — Express Backend

---

📍 **Progress Marker**
- Last Revised: ___________
- Revision Count: 0
- Difficulty Level: 🔵 Medium–Hard

---

## Q1 — How does Express middleware work? Explain the middleware pipeline.



## 🚀 What is Middleware?

Middleware in Express is a function that executes **between request and response**.

It has access to:

- `req` → request
    
- `res` → response
    
- `next` → function to pass control
    

👉 It forms a **pipeline (chain of functions)**.

---

## 🔄 Middleware Pipeline

```
Request
  ↓
Middleware 1
  ↓
Middleware 2
  ↓
Route Handler
  ↓
Response
```

### Key Rules:

- Executes **in order**
    
- Must call `next()` OR send response
    
- If neither → request hangs ❌
    

---

## 🧩 Basic Example

```js
app.use((req, res, next) => {
  console.log("Middleware 1");
  next();
});

app.use((req, res, next) => {
  console.log("Middleware 2");
  next();
});

app.get("/", (req, res) => {
  res.send("Hello World");
});
```

---

# 📚 Types of Middleware

---

## 1️⃣ Application-Level Middleware

### 📌 Definition:

Attached to `app` → runs globally or for specific path

```js
app.use((req, res, next) => {
  console.log("Global middleware");
  next();
});
```

```js
app.use("/api", middleware); // only for /api/*
```

### 🎯 Use Cases:

- Logging
    
- Authentication
    
- Parsing
    

---

## 2️⃣ Router-Level Middleware

### 📌 Definition:

Applied to a specific router

```js
const router = express.Router();

router.use((req, res, next) => {
  console.log("Router middleware");
  next();
});

app.use("/user", router);
```

### 🎯 Use Cases:

- Modular structure
    
- Feature-based routing
    

---

## 3️⃣ Built-in Middleware

### 📌 Provided by Express

```js
app.use(express.json());
app.use(express.urlencoded({ extended: true }));
app.use(express.static("public"));
```

### 🎯 Purpose:

- Parse JSON
    
- Parse form data
    
- Serve static files
    

---

## 4️⃣ Third-Party Middleware

### 📌 External libraries

```js
const morgan = require("morgan");
app.use(morgan("dev"));
```

### 🎯 Examples:

- Logging
    
- Security
    
- CORS
    
- Rate limiting
    

---

## 5️⃣ Route-Specific Middleware

### 📌 Applied to a single route

```js
app.get("/dashboard", authMiddleware, (req, res) => {
  res.send("Dashboard");
});
```

### 🎯 Use Cases:

- Authentication
    
- Validation
    

---

## 6️⃣ Error-Handling Middleware ⚠️

### 📌 Special Middleware (4 params)

```js
app.use((err, req, res, next) => {
  console.error(err.message);
  res.status(500).send("Something went wrong");
});
```

### ⚙️ Behavior:

- Triggered via `next(err)`
    
- Skips normal middleware
    

---

# 🔥 Execution Flow (Big Picture)

```
Request
  ↓
App-level middleware
  ↓
Router-level middleware
  ↓
Route-specific middleware
  ↓
Route handler
  ↓
Error middleware (if error)
  ↓
Response
```

---

# ⚠️ Important Interview Points

- Order matters 🔑
    
- `next()` is mandatory (unless response sent)
    
- Middleware can:
    
    - Modify req/res
        
    - End response
        
    - Pass control
        

---

# 🧠 One-Line Summary

👉 Middleware is a chain of functions that process a request step-by-step using `next()` to move through the pipeline.

---

# 💡 Bonus (Pro Insight)

Internally, Express maintains a **stack of middleware layers**, and `next()` simply moves to the next layer in that stack.

---

## Q2 — How does Express error handling work?

## 📌 Overview

Error handling in Express.js works in **layers**, not just one mechanism.

```
1. try...catch
2. Promise .catch()
3. next(err) → Express Error Middleware
4. process.on (last fallback)
```

---

# 1️⃣ try...catch (First Line of Defense)

## ✅ Works for:

- Synchronous code
    
- `await` async code
    

```js
try {
  const data = await getData();
  res.send(data);
} catch (err) {
  next(err);
}
```

---

## ❌ Important Edge Case (VERY IMPORTANT ⚠️)

```js
try {
  setTimeout(() => {
    throw new Error("Boom"); // ❌ NOT caught
  }, 1000);
} catch (err) {
  console.log("Will not run");
}
```

### 💥 Why this fails:

- `setTimeout` runs in a **different call stack (event loop)**
    
- `try...catch` only catches errors in the **same synchronous execution context**
    

---

## ✅ Correct Way to Handle

```js
setTimeout(() => {
  try {
    throw new Error("Boom");
  } catch (err) {
    console.log("Caught:", err.message);
  }
}, 1000);
```

OR (better in Express):

```js
setTimeout(() => {
  next(new Error("Boom"));
}, 1000);
```

---

# 2️⃣ Promise `.catch()` (Async Handling)

```js
getData()
  .then(data => res.send(data))
  .catch(err => next(err));
```

👉 Equivalent to try/catch for promises

---

# 3️⃣ Express Error Middleware (Centralized)

```js
app.use((err, req, res, next) => {
  res.status(500).json({
    message: err.message
  });
});
```

## 📌 Rules:

- Must have **4 params** → `(err, req, res, next)`
    
- Must be placed **after all routes**
    

---

# 🔥 Async Wrapper (Clean Pattern)

```js
const asyncHandler = fn => (req, res, next) =>
  Promise.resolve(fn(req, res, next)).catch(next);
```

Usage:

```js
app.get('/', asyncHandler(async (req, res) => {
  const data = await getData();
  res.send(data);
}));
```

---

# 4️⃣ process.on (Last Safety Net 🚨)

## Uncaught Exceptions

```js
process.on('uncaughtException', err => {
  console.error(err);
  process.exit(1);
});
```

## Unhandled Promise Rejections

```js
process.on('unhandledRejection', err => {
  console.error(err);
  process.exit(1);
});
```

---

## ⚠️ Important:

- This is **NOT error handling**, it's **damage control**
    
- App may be in **corrupted state**
    
- Best practice → **log + restart**
    

---

# 🧩 Full Flow (Real World)

```
Route
 ↓
try/catch OR .catch()
 ↓
next(err)
 ↓
Express Error Middleware
 ↓
(if missed)
process.on()
```

---

# 🚀 Additional Important Points (Often Missed)

## ✅ 1. Always `return` after sending response

```js
if (!user) {
  return next(new Error("User not found")); // prevent further execution
}
```

---

## ✅ 2. Avoid multiple responses

```js
res.send("A");
res.send("B"); // ❌ crash
```

---

## ✅ 3. Custom Error Class (Production Standard)

```js
class AppError extends Error {
  constructor(message, statusCode) {
    super(message);
    this.statusCode = statusCode;
  }
}
```

---

## ✅ 4. Express 5 Insight (Advanced 🔥)

- Automatically catches async errors
    
- No need for wrapper like `asyncHandler`
    

---

# 🎯 Interview Summary (Quick Recall)

- `try...catch` → sync + awaited async only
    
- ❌ Fails for `setTimeout`, event loop async
    
- `.catch()` → handles promises
    
- `next(err)` → passes error to Express
    
- Error middleware → centralized handling
    
- `process.on` → last fallback, restart app
    

---

# 🧠 One-Liner (Senior Level)

> "Errors in Node.js must be handled at the same async boundary; otherwise they escape to the event loop and require higher-level handlers like Express middleware or process-level fallbacks."


# 🧠 `throw` vs `Error` (Short Notes)

---

## 🔹 `throw`

- Keyword used to **raise an exception**
    
- Stops execution and jumps to nearest `catch`
    

```js
throw new Error("Something went wrong");
```

---

## 🔹 `Error`

- Built-in class to create **error objects**
    
- Provides:
    
    - `message`
        
    - `name`
        
    - `stack` ⭐ (debugging)
        

```js
const err = new Error("Invalid input");
```

---

## 🔹 Together (Best Practice)

```js
throw new Error("Invalid input");
```

- `throw` → triggers error
    
- `Error` → gives structure + stack trace
    

---

## 🔹 Async Behavior ⚠️

- `try/catch` works only for **sync**
    
- Async → use:
    
    - `async/await + try/catch`
        
    - `.catch()`
        

---

## 🔹 Best Practices

- ✅ Always throw `Error` (not string/number)
    
- ✅ Use custom errors in backend
    
- ❌ Don’t lose stack (`throw err`, not `throw new Error(err)`)
    
---

## Q3 — What is the Express request lifecycle?

### ✅ Simple Explanation
A request in Express flows: incoming HTTP request → global middleware → router → route-specific middleware → route handler → response sent.

### 🧠 Deep Dive
```
HTTP Request
    │
    ▼
[express.json()]        ← body parsing
[cors()]                ← CORS headers
[morgan()]              ← logging
[helmet()]              ← security headers
    │
    ▼
[Router matching]       ← app.use('/api', router)
    │
    ▼
[Route middleware]      ← requireAuth, validate
    │
    ▼
[Route handler]         ← async (req, res) => {}
    │
    ├── res.json()      ← response sent
    └── next(err)       ← error handler
            │
            ▼
    [Error handler]     ← (err, req, res, next)
```

**`req` properties you should know:**
- `req.params` — URL params (`:id`)
- `req.query` — query string (`?page=2`)
- `req.body` — parsed body (needs `express.json()`)
- `req.headers` — HTTP headers
- `req.ip` — client IP (use `trust proxy` for real IP behind load balancer)

### 💻 Code Example
```js
// Full production Express setup
const express = require('express');
const cors = require('cors');
const helmet = require('helmet');
const morgan = require('morgan');
const rateLimit = require('express-rate-limit');

const app = express();

// Trust proxy (for reverse proxy like Nginx)
app.set('trust proxy', 1);

// Security middleware
app.use(helmet());
app.use(cors({
  origin: process.env.CLIENT_URL,
  credentials: true
}));

// Rate limiting
const limiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 100
});
app.use('/api', limiter);

// Body parsing
app.use(express.json({ limit: '10kb' })); // Limit payload size
app.use(express.urlencoded({ extended: false }));

// Logging
if (process.env.NODE_ENV !== 'test') {
  app.use(morgan('combined'));
}

// Routes
app.use('/api/users', userRouter);
app.use('/api/todos', todoRouter);

// 404
app.all('*', (req, res, next) => {
  next(new AppError(`${req.method} ${req.path} not found`, 404));
});

// Error handler (always last)
app.use(errorHandler);
```

### ⚠️ Common Mistakes
- Not setting `app.set('trust proxy', 1)` behind Nginx — `req.ip` returns wrong IP
- Setting payload limit too high — opens DoS vulnerability
- Registering routes after error handler

### 🎯 Interview Tip
> "I set up security middleware (helmet, cors, rate limiting) first, then body parsers, then routes. Error handler always goes last."

---

## Q4 — How do you structure a production Express application?

### ✅ Simple Explanation
Use a feature-based or layer-based folder structure with separation of concerns: routes → controllers → services → models.

### 🧠 Deep Dive
**Layer-based architecture:**
```
src/
├── app.js              ← Express setup, middleware
├── server.js           ← HTTP server, DB connection
├── routes/
│   ├── user.routes.js
│   └── todo.routes.js
├── controllers/
│   ├── user.controller.js   ← req/res handling only
│   └── todo.controller.js
├── services/
│   ├── user.service.js      ← business logic
│   └── todo.service.js
├── models/
│   ├── user.model.js        ← DB schema
│   └── todo.model.js
├── middleware/
│   ├── auth.js
│   └── validate.js
├── utils/
│   └── AppError.js
└── config/
    └── db.js
```

**Why separate server.js from app.js?**
- `app.js` = Express app (testable without starting server)
- `server.js` = starts HTTP server, DB connection, process listeners

### 💻 Code Example
```js
// routes/todo.routes.js
const router = require('express').Router();
const { getTodos, createTodo } = require('../controllers/todo.controller');
const { requireAuth } = require('../middleware/auth');
const { validateTodo } = require('../middleware/validate');

router.get('/', requireAuth, getTodos);
router.post('/', requireAuth, validateTodo, createTodo);
module.exports = router;

// controllers/todo.controller.js
const todoService = require('../services/todo.service');
const asyncHandler = require('../utils/asyncHandler');

exports.getTodos = asyncHandler(async (req, res) => {
  const todos = await todoService.getUserTodos(req.user.id);
  res.json({ status: 'success', data: todos });
});

// services/todo.service.js
const Todo = require('../models/todo.model');

exports.getUserTodos = async (userId) => {
  return Todo.find({ user: userId }).sort('-createdAt');
};

// server.js
const app = require('./app');
const connectDB = require('./config/db');

const PORT = process.env.PORT || 8000;

async function start() {
  await connectDB();
  const server = app.listen(PORT, () => {
    console.log(`Server running on port ${PORT}`);
  });

  process.on('SIGTERM', () => {
    server.close(() => process.exit(0));
  });
}
start();
```

### ⚠️ Common Mistakes
- Putting business logic in controllers — makes testing hard
- Not separating `app.js` from `server.js` — can't unit test Express app

### 🎯 Interview Tip
> "Controllers handle HTTP concerns only (req/res). Business logic belongs in services. This separation makes code testable and follows single responsibility principle."

---

## Q5 — How do you validate request data in Express?

## 📌 What is Zod?

- TypeScript-first **schema validation library**
    
- Used for **runtime validation + data parsing**
    
- Ensures incoming data is **safe & structured**
    

---

## ⚡ Why use Zod in Express?

- Validate:
    
    - `req.body`
        
    - `req.query`
        
    - `req.params`
        
- Prevent **invalid/untrusted data**
    
- Avoid manual validation logic
    

---

## 🧠 Core Concept

```js
const schema = z.object({
  email: z.string().email(),
});

schema.parse(data);      // ❌ throws error
schema.safeParse(data);  // ✅ safe (recommended)
```

---

## 🚀 Express Integration Flow

```
Request
 → Validate (Zod Middleware)
 → Controller
 → Service
 → Response
 → Error Middleware
```

---

## 🧱 Basic Schema

```js
const { z } = require("zod");

const userSchema = z.object({
  email: z.string().email(),
  password: z.string().min(6),
});
```

---

## 🔥 Validate Middleware (Reusable)

```js
const validate = (schema) => (req, res, next) => {
  const result = schema.safeParse({
    body: req.body,
    query: req.query,
    params: req.params,
  });

  if (!result.success) return next(result.error);

  req.body = result.data.body;
  req.query = result.data.query;
  req.params = result.data.params;

  next();
};
```

---

## 🛣️ Route Usage

```js
router.post("/signup", validate(schema), controller);
```

---

## ❗ Global Error Handler

```js
if (err.name === "ZodError") {
  return res.status(400).json(err.errors);
}
```

---

## 📦 Schema Structure (Best Practice)

```js
const schema = z.object({
  body: z.object({...}),
  query: z.object({...}),
  params: z.object({...}),
});
```

---

## 🔧 Common Validators

```js
z.string()
z.string().email()
z.string().min(6)
z.number().int()
z.array(z.string())
```

---

## 🔄 Useful Methods

```js
.safeParse()   // safe validation
.parse()       // throws error

.transform()   // modify value
.refine()      // custom validation
.strict()      // no extra fields
```

---

## 🧼 Data Sanitization

```js
z.string().transform(val => val.trim().toLowerCase());
```

---

## ⚠️ Important Rules

- Always use `.safeParse()` in APIs
    
- Always use **middleware (not controller)**
    
- Always use **validated data (`result.data`)**
    
- Keep validation **separate from business logic**
    

---

## ❌ Common Mistakes

- Validation inside controller ❌
    
- Using `.parse()` in API ❌
    
- No global error handler ❌
    
- Not sanitizing input ❌
    

---

## 💡 JS vs TS

- Works in **JavaScript ✅**
    
- Type inference only in **TypeScript**
    

---

## 🧠 Interview One-Liner

**Zod is used as a middleware-based validation layer in Express to validate and sanitize incoming requests before reaching controllers, with centralized error handling.**

---

## ⚡ Quick Revision Summary

- Middleware is a linked-list pipeline; `next()` advances, `next(err)` jumps to error handler
- Error middleware must have exactly 4 params `(err, req, res, next)` and be registered LAST
- In Express 4, async route handlers must be wrapped; Express 5 auto-catches async errors
- Use `helmet`, `cors`, rate limiting, body size limits for production security
- Layer structure: route → controller (HTTP only) → service (business logic) → model
- Always validate at the request boundary before controllers

---

## 🏆 Top 5 Must Revise Before Interview

1. **Middleware execution order** — global → router → route-specific
2. **Async error handling in Express 4** — `asyncHandler` wrapper pattern
3. **Error middleware 4-param rule** — most candidates miss this
4. **Request lifecycle** — draw the flow from HTTP request to response
5. **Controller vs Service separation** — architectural principle

---

## 🎤 Real Interview Questions

- *"What happens if you forget to call `next()` in middleware?"*
- *"How do you handle async errors in Express 4?"*
- *"Where would you add rate limiting in your Express app?"*
- *"What's the difference between `app.use` and `router.use`?"*
- *"How would you structure a large Express application?"*
