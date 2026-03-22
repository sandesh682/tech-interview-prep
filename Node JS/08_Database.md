# ⚪ 08 — Database with Node.js

---

📍 **Progress Marker**
- Last Revised: ___________
- Revision Count: 0
- Difficulty Level: ⚪ Medium–Hard

---

## Q1 — How does Mongoose work? Explain schema, model, and validation.

### ✅ Simple Explanation
Mongoose is an ODM (Object Data Mapper) for MongoDB. It adds schemas, validation, middleware hooks, and query helpers on top of the MongoDB driver.

### 🧠 Deep Dive
**Key concepts:**
- **Schema**: defines structure, types, validators, defaults
- **Model**: compiled schema + MongoDB collection interface
- **Document**: instance of a model (a row)
- **Virtuals**: computed properties not stored in DB
- **Middleware (hooks)**: `pre`/`post` hooks on operations

**Schema types:** `String`, `Number`, `Boolean`, `Date`, `ObjectId`, `Array`, `Mixed`, `Map`

### 💻 Code Example
```js
const mongoose = require('mongoose');

// Schema with validation
const userSchema = new mongoose.Schema({
  name: {
    type: String,
    required: [true, 'Name is required'],
    trim: true,
    maxlength: [50, 'Name too long'],
  },
  email: {
    type: String,
    required: true,
    unique: true,
    lowercase: true,
    validate: {
      validator: (v) => /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(v),
      message: 'Invalid email format',
    },
  },
  password: {
    type: String,
    required: true,
    minlength: 8,
    select: false, // never returned in queries by default
  },
  role: {
    type: String,
    enum: ['user', 'admin'],
    default: 'user',
  },
  createdAt: { type: Date, default: Date.now },
}, {
  timestamps: true, // auto createdAt + updatedAt
  toJSON: { virtuals: true },
});

// Virtual (not stored in DB)
userSchema.virtual('fullProfile').get(function() {
  return `${this.name} (${this.email})`;
});

// Pre-save hook — hash password
userSchema.pre('save', async function(next) {
  if (!this.isModified('password')) return next();
  this.password = await bcrypt.hash(this.password, 12);
  next();
});

// Instance method
userSchema.methods.comparePassword = async function(candidate) {
  return bcrypt.compare(candidate, this.password);
};

// Static method
userSchema.statics.findByEmail = function(email) {
  return this.findOne({ email: email.toLowerCase() });
};

const User = mongoose.model('User', userSchema);

// CRUD operations
const user = await User.create({ name: 'Sandesh', email: 'sandesh@example.com', password: 'secret' });
const users = await User.find({ role: 'admin' }).select('name email').lean();
const updated = await User.findByIdAndUpdate(id, { name: 'New Name' }, { new: true, runValidators: true });
await User.findByIdAndDelete(id);
```

### ⚠️ Common Mistakes
- Not using `{ new: true }` in `findByIdAndUpdate` — returns OLD document
- Forgetting `runValidators: true` on update — validators don't run by default on updates
- Using `save()` in a loop — triggers hooks N times; use `insertMany` for bulk
- Not using `.lean()` on read-only queries — returns plain objects (10x faster for large datasets)

### 🎯 Interview Tip
> "For read-heavy routes, I chain `.lean()` on queries — it skips Mongoose document wrapping and returns plain JS objects, which is significantly faster."

---

## Q2 — How does indexing work in MongoDB and when should you add indexes?

### ✅ Simple Explanation
An index is a data structure that makes queries faster by avoiding full collection scans. Without an index, MongoDB scans every document (COLLSCAN). With an index, it's a B-tree lookup (IXSCAN).

### 🧠 Deep Dive
**Index types:**
- **Single field**: `{ email: 1 }` — ascending
- **Compound**: `{ userId: 1, createdAt: -1 }` — multi-field
- **Text**: `{ title: 'text' }` — full-text search
- **Sparse**: only indexes documents that have the field
- **Partial**: indexes documents matching a filter condition
- **TTL**: auto-deletes documents after a time (logs, sessions)

**When to add indexes:**
- Fields used in `find()`, `sort()`, `groupBy`
- Fields with high cardinality (many unique values)
- Foreign key fields (userId in a posts collection)

**Index downsides:**
- Write overhead (index updated on every insert/update/delete)
- Memory usage
- Don't over-index

### 💻 Code Example
```js
// In Mongoose schema
const todoSchema = new mongoose.Schema({
  userId: { type: ObjectId, ref: 'User', index: true }, // single field index
  title: String,
  status: { type: String, enum: ['pending', 'done'] },
  createdAt: { type: Date, default: Date.now },
});

// Compound index (most todos will filter by user + sort by date)
todoSchema.index({ userId: 1, createdAt: -1 });

// Partial index — only index pending todos (where status='pending')
todoSchema.index(
  { userId: 1 },
  { partialFilterExpression: { status: 'pending' } }
);

// TTL index — auto-delete sessions after 1 hour
const sessionSchema = new mongoose.Schema({
  token: String,
  createdAt: { type: Date, default: Date.now },
});
sessionSchema.index({ createdAt: 1 }, { expireAfterSeconds: 3600 });

// Text index for search
todoSchema.index({ title: 'text', description: 'text' });
const results = await Todo.find({ $text: { $search: 'important meeting' } });

// Check query plan
const plan = await Todo.find({ userId: someId }).explain('executionStats');
console.log(plan.executionStats.executionStages.stage); // IXSCAN = good, COLLSCAN = bad
```

### ⚠️ Common Mistakes
- Adding indexes in production on a large collection without building in background
- Indexing every field — wastes write performance and memory
- Not having index on foreign key fields — `populate()` causes COLLSCAN

### 🎯 Interview Tip
> "I use `.explain('executionStats')` to check if my query is using an index. If I see `COLLSCAN` and the collection is large, I add an index. For compound indexes, field order matters — put equality checks first, then range/sort fields."

---

## Q3 — What are MongoDB aggregation pipelines and when do you use them?

### ✅ Simple Explanation
Aggregation pipelines process documents through multiple stages to compute results — like SQL's GROUP BY, JOIN, HAVING. Use them for analytics, reporting, and complex data transformations.

### 🧠 Deep Dive
**Common stages:**
- `$match` — filter documents (like WHERE)
- `$group` — group and aggregate (like GROUP BY)
- `$project` — reshape documents (include/exclude/transform fields)
- `$lookup` — join with another collection (like LEFT JOIN)
- `$unwind` — deconstruct array fields
- `$sort` — sort results
- `$limit` / `$skip` — pagination
- `$addFields` — add computed fields

**Performance tip:** `$match` and `$project` early in pipeline — reduces documents for later stages.

### 💻 Code Example
```js
// Example: Get todo stats per user for the last 30 days
const stats = await Todo.aggregate([
  // Stage 1: Filter
  {
    $match: {
      createdAt: { $gte: new Date(Date.now() - 30 * 24 * 3600 * 1000) }
    }
  },
  // Stage 2: Group by user
  {
    $group: {
      _id: '$userId',
      total: { $sum: 1 },
      completed: { $sum: { $cond: [{ $eq: ['$status', 'done'] }, 1, 0] } },
      pending: { $sum: { $cond: [{ $eq: ['$status', 'pending'] }, 1, 0] } },
      avgTitle: { $avg: { $strLenCP: '$title' } }
    }
  },
  // Stage 3: Join with users collection
  {
    $lookup: {
      from: 'users',
      localField: '_id',
      foreignField: '_id',
      as: 'user',
      pipeline: [{ $project: { name: 1, email: 1 } }] // only get needed fields
    }
  },
  // Stage 4: Unwind (array → single object)
  { $unwind: '$user' },
  // Stage 5: Project final shape
  {
    $project: {
      userName: '$user.name',
      userEmail: '$user.email',
      total: 1,
      completed: 1,
      completionRate: {
        $multiply: [{ $divide: ['$completed', '$total'] }, 100]
      }
    }
  },
  { $sort: { completionRate: -1 } },
  { $limit: 10 }
]);
```

### ⚠️ Common Mistakes
- Not putting `$match` first — processes all documents unnecessarily
- Using `$lookup` without limiting fields in the joined collection (`pipeline` option)
- Using aggregation when a simple query works — adds complexity

### 🎯 Interview Tip
> "For analytical queries spanning multiple collections or needing computed fields, I use aggregation. For simple CRUD, Mongoose's query methods are cleaner and faster."

---

## Q4 — How do you handle database connections properly in Node.js?

### ✅ Simple Explanation
Connect once at app startup, handle connection errors, reconnect automatically, and close gracefully on shutdown.

### 🧠 Deep Dive
**Connection lifecycle:**
1. App starts → `mongoose.connect()` → pool created
2. Requests use connections from pool
3. App shuts down → `mongoose.disconnect()` → all connections closed

**Mongoose connection events:** `connected`, `error`, `disconnected`, `reconnected`

**Never reconnect per-request** — it's not a per-request operation.

### 💻 Code Example
```js
// config/db.js
const mongoose = require('mongoose');

const connectDB = async () => {
  try {
    const conn = await mongoose.connect(process.env.MONGODB_URI, {
      maxPoolSize: 10,
      minPoolSize: 2,
      serverSelectionTimeoutMS: 5000, // fail fast if no server
      socketTimeoutMS: 45000,
      heartbeatFrequencyMS: 10000,
    });

    console.log(`MongoDB connected: ${conn.connection.host}`);

    mongoose.connection.on('error', (err) => {
      console.error('MongoDB error:', err);
    });

    mongoose.connection.on('disconnected', () => {
      console.warn('MongoDB disconnected. Reconnecting...');
    });

  } catch (err) {
    console.error('DB connection failed:', err.message);
    process.exit(1); // can't start without DB
  }
};

// Graceful shutdown
const gracefulShutdown = async (signal) => {
  console.log(`${signal} received. Closing DB connections...`);
  await mongoose.disconnect();
  console.log('MongoDB disconnected. Process exiting.');
  process.exit(0);
};

process.on('SIGTERM', () => gracefulShutdown('SIGTERM'));
process.on('SIGINT', () => gracefulShutdown('SIGINT'));

module.exports = connectDB;
```

### ⚠️ Common Mistakes
- Not calling `mongoose.disconnect()` on shutdown — open connections prevent clean process exit
- Not exiting process on DB connection failure — app starts in broken state
- Opening a new connection inside a request handler

### 🎯 Interview Tip
> "I connect once at startup with `maxPoolSize` tuned to my load. I listen to connection events for health monitoring, and I always handle graceful shutdown — critical for Kubernetes deployments."

---

## Q5 — What is the N+1 query problem and how do you fix it?

### ✅ Simple Explanation
N+1 means: 1 query to get N items, then N separate queries to get related data for each. It kills performance at scale.

### 🧠 Deep Dive
Classic example: get 100 users, then fetch each user's todos separately = 101 queries.

**Fixes:**
1. **Mongoose populate** — still does 2 queries but batched (not N+1)
2. **Aggregation $lookup** — single pipeline
3. **DataLoader pattern** — batches & caches (used in GraphQL)

### 💻 Code Example
```js
// ❌ N+1 problem
const users = await User.find({}); // 1 query
for (const user of users) {
  user.todos = await Todo.find({ userId: user._id }); // N queries!
}

// ✅ Fix 1: populate (2 queries, not N+1)
const users = await User.find({}).populate('todos');

// ✅ Fix 2: aggregation $lookup (1 pipeline)
const usersWithTodos = await User.aggregate([
  { $lookup: {
    from: 'todos',
    localField: '_id',
    foreignField: 'userId',
    as: 'todos'
  }}
]);

// ✅ Fix 3: Manual batch query
const users = await User.find({});
const userIds = users.map(u => u._id);
const allTodos = await Todo.find({ userId: { $in: userIds } });

// Group todos by userId in memory (O(n) not O(n²))
const todosByUser = allTodos.reduce((acc, todo) => {
  const key = todo.userId.toString();
  acc[key] = acc[key] || [];
  acc[key].push(todo);
  return acc;
}, {});

const result = users.map(user => ({
  ...user.toObject(),
  todos: todosByUser[user._id.toString()] || []
}));
```

### ⚠️ Common Mistakes
- Calling `populate` inside a loop — equivalent to N+1
- Using `populate` for large collections — aggregation `$lookup` with projection is more efficient

### 🎯 Interview Tip
> "N+1 is one of the most common performance bugs. I spot it in code reviews by looking for `find` or `findById` calls inside loops. Fix with `$in` queries or `$lookup`."

---

## ⚡ Quick Revision Summary

- Mongoose: schema → model → document; `.lean()` for read-heavy queries
- `{ new: true, runValidators: true }` on update operations
- Indexes: single, compound, text, TTL; use `explain()` to verify IXSCAN
- Aggregation: `$match` early, `$lookup` for joins, `$group` for analytics
- Connection: once at startup, graceful shutdown with `SIGTERM`
- N+1: fix with `populate()`, `$lookup`, or `$in` batch queries

---

## 🏆 Top 5 Must Revise Before Interview

1. **Mongoose hooks** — pre-save password hashing is a classic
2. **Indexing basics** — compound index field order, `explain()` usage
3. **N+1 query problem** — recognition and all three fixes
4. **Aggregation pipeline** — know `$match`, `$group`, `$lookup`, `$project`
5. **Connection pool sizing** — formula and graceful shutdown

---

## 🎤 Real Interview Questions

- *"What is the N+1 problem? How would you detect it in a Node.js app?"*
- *"How does `populate` work in Mongoose? What SQL operation is it like?"*
- *"When would you add an index to a MongoDB collection? What are the downsides?"*
- *"Explain `.lean()` in Mongoose and when you'd use it."*
