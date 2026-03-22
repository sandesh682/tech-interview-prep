# 🔵 04 — Express Backend

---

📍 **Progress Marker**
- Last Revised: ___________
- Revision Count: 0
- Difficulty Level: 🔵 Medium–Hard

---

## Q1 — How does Express middleware work? Explain the middleware pipeline.

### ✅ Simple Explanation
Middleware are functions that have access to `req`, `res`, and `next`. They form a pipeline — each middleware either responds to the request or passes control to the next one via `next()`.

### 🧠 Deep Dive
**Middleware signature:** `(req, res, next) => void`

**Types of middleware:**
- Application-level: `app.use(fn)`
- Router-level: `router.use(fn)`
- Error-handling: `(err, req, res, next) => void` — 4 params
- Built-in: `express.json()`, `express.static()`
- Third-party: `cors()`, `helmet()`, `morgan()`

**Execution order:** Middleware runs in the order it's registered. `app.use()` without a path matches all routes.

**`next('route')`** — skip remaining handlers in current route, move to next matching route.

### 💻 Code Example
```js
const express = require('express');
const app = express();

// 1. Built-in middleware
app.use(express.json()); // parse JSON body
app.use(express.urlencoded({ extended: true })); // parse form data

// 2. Custom logger middleware
app.use((req, res, next) => {
  const start = Date.now();
  res.on('finish', () => {
    console.log(`${req.method} ${req.path} ${res.statusCode} ${Date.now() - start}ms`);
  });
  next(); // MUST call next() or request hangs
});

// 3. Auth middleware
function requireAuth(req, res, next) {
  const token = req.headers.authorization?.split(' ')[1];
  if (!token) return res.status(401).json({ error: 'Unauthorized' });
  try {
    req.user = jwt.verify(token, process.env.JWT_SECRET);
    next();
  } catch {
    res.status(401).json({ error: 'Invalid token' });
  }
}

// 4. Protected route
app.get('/profile', requireAuth, (req, res) => {
  res.json({ user: req.user });
});

// 5. Error-handling middleware (4 params — MUST be last)
app.use((err, req, res, next) => {
  console.error(err.stack);
  res.status(err.statusCode || 500).json({
    error: err.message || 'Internal Server Error'
  });
});
```

### ⚠️ Common Mistakes
- Forgetting to call `next()` — request hangs indefinitely
- Placing error-handling middleware before routes — it never gets called
- Not having 4 params in error middleware — Express won't treat it as error handler

### 🎯 Interview Tip
> "Middleware is just a function in a linked list. When you call `next()`, it moves to the next function. When you call `next(err)`, it jumps to the error-handling middleware."

---

## Q2 — How does Express error handling work?

### ✅ Simple Explanation
Errors are forwarded to error-handling middleware by calling `next(err)`. Error middleware has 4 parameters: `(err, req, res, next)`.

### 🧠 Deep Dive
**Sync errors:** Express catches them automatically in route handlers (Express 5), but in Express 4 you must catch them yourself.

**Async errors in Express 4:** You MUST catch and call `next(err)` manually. Express 4 does NOT catch async errors automatically.

**Custom error class:** Create domain-specific errors with HTTP status codes.

### 💻 Code Example
```js
// Custom error class
class AppError extends Error {
  constructor(message, statusCode) {
    super(message);
    this.statusCode = statusCode;
    this.isOperational = true; // vs programmer errors
  }
}

// Express 4 — MUST wrap async handlers
const asyncHandler = (fn) => (req, res, next) => {
  Promise.resolve(fn(req, res, next)).catch(next);
};

// Usage
app.get('/user/:id', asyncHandler(async (req, res) => {
  const user = await User.findById(req.params.id);
  if (!user) throw new AppError('User not found', 404);
  res.json(user);
}));

// Express 5 — async errors caught automatically (no wrapper needed)
// app.get('/user/:id', async (req, res) => { ... throw ... });

// Centralized error handler
app.use((err, req, res, next) => {
  // Log error
  if (!err.isOperational) {
    console.error('PROGRAMMER ERROR:', err);
    // Consider restarting process for non-operational errors
  }

  const statusCode = err.statusCode || 500;
  const message = err.isOperational ? err.message : 'Something went wrong';

  res.status(statusCode).json({
    status: 'error',
    message,
    ...(process.env.NODE_ENV === 'development' && { stack: err.stack })
  });
});

// Handle 404 — add before error handler, after all routes
app.use((req, res, next) => {
  next(new AppError(`Route ${req.path} not found`, 404));
});
```

### ⚠️ Common Mistakes
- Not wrapping async route handlers in Express 4 — uncaught rejections crash the process
- Not differentiating operational errors (404, validation) from programmer errors (null refs)

### 🎯 Interview Tip
> "In Express 4, I always use an `asyncHandler` wrapper. In Express 5 this is fixed. I also distinguish operational errors (user-facing) from programmer errors (bugs) — programmer errors should potentially restart the process."

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

### ✅ Simple Explanation
Use a validation library like `joi`, `zod`, or `express-validator` in middleware before reaching the route handler.

### 🧠 Deep Dive
**Options:**
- `joi` — schema-based, mature, server-side focused
- `zod` — TypeScript-first, works in Node too, excellent DX
- `express-validator` — decorator-style, chains

Validation should happen in middleware so controllers receive clean, trusted data.

### 💻 Code Example
```js
// Using zod (modern approach)
const { z } = require('zod');

const createTodoSchema = z.object({
  body: z.object({
    title: z.string().min(1).max(100),
    description: z.string().optional(),
    priority: z.enum(['low', 'medium', 'high']).default('medium'),
  })
});

// Generic validation middleware factory
const validate = (schema) => (req, res, next) => {
  try {
    schema.parse({ body: req.body, query: req.query, params: req.params });
    next();
  } catch (err) {
    if (err instanceof z.ZodError) {
      return res.status(400).json({
        status: 'error',
        errors: err.errors.map(e => ({
          field: e.path.join('.'),
          message: e.message
        }))
      });
    }
    next(err);
  }
};

// Route with validation
router.post('/todos', requireAuth, validate(createTodoSchema), createTodo);
```

### ⚠️ Common Mistakes
- Validating in controllers — code smell, mixes concerns
- Trusting client-sent data without sanitization (XSS, injection risk)
- Not validating `req.params` and `req.query` — only validating body

### 🎯 Interview Tip
> "I always validate at the boundary — before business logic runs. Zod is my preference for its TypeScript inference and composability."

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
