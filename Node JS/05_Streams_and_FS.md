# 🟣 05 — Streams and File System

---

📍 **Progress Marker**
- Last Revised: ___________
- Revision Count: 0
- Difficulty Level: 🟣 Medium–Hard

---

## Q1 — What are Streams in Node.js and why are they important?

### ✅ Simple Explanation
Streams are objects that let you read or write data piece by piece (chunks), rather than loading the entire data into memory. They're essential for processing large files, network data, or any continuous data source efficiently.

### 🧠 Deep Dive
**4 types of streams:**
| Type | Description | Example |
|---|---|---|
| Readable | Data source you can read from | `fs.createReadStream`, HTTP request |
| Writable | Destination you can write to | `fs.createWriteStream`, HTTP response |
| Duplex | Both readable and writable | TCP socket, `net.Socket` |
| Transform | Duplex that transforms data | `zlib.createGzip()`, crypto streams |

**Key concepts:**
- `pipe()`: connects Readable → Writable, handles backpressure
- **Backpressure**: when writer is slower than reader — pipe handles this automatically
- Streams use EventEmitter: `data`, `end`, `error`, `finish` events
- **Highwater mark**: buffer size (default 16KB for bytes, 16 objects for object mode)

### 💻 Code Example
```js
const fs = require('fs');
const zlib = require('zlib');
const { pipeline } = require('stream/promises'); // Node 15+

// ❌ Memory-intensive for large files
const data = fs.readFileSync('./large-file.csv'); // Loads ENTIRE file
process(data);

// ✅ Stream — processes chunk by chunk
const readStream = fs.createReadStream('./large-file.csv');
readStream.on('data', (chunk) => {
  process(chunk); // chunk is a Buffer
});
readStream.on('end', () => console.log('Done'));
readStream.on('error', (err) => console.error(err));

// ✅ Best: pipeline with error handling
async function compressFile(input, output) {
  await pipeline(
    fs.createReadStream(input),
    zlib.createGzip(),
    fs.createWriteStream(output)
  );
  console.log('Compressed successfully');
}

// HTTP streaming response
app.get('/large-file', (req, res) => {
  res.setHeader('Content-Type', 'application/octet-stream');
  const stream = fs.createReadStream('./large-file.zip');
  stream.pipe(res); // backpressure handled automatically
  stream.on('error', (err) => res.destroy(err));
});
```

### ⚠️ Common Mistakes
- Using `.pipe()` instead of `pipeline()` — `.pipe()` doesn't forward errors from source to destination
- Not handling `error` event on streams — unhandled stream errors crash the process
- Forgetting that `chunk` is a `Buffer` by default — use `.toString()` or set encoding

### 🎯 Interview Tip
> "Use `stream/promises pipeline()` over `.pipe()` in production — it properly handles errors and cleanup across all streams in the chain."

---

## Q2 — What is backpressure and how does Node handle it?

### ✅ Simple Explanation
Backpressure occurs when data is produced faster than it can be consumed. Without backpressure handling, the memory buffer grows unbounded and can crash your process.

### 🧠 Deep Dive
**How pipe handles backpressure:**
1. `readable.pipe(writable)` listens to `drain` event
2. When writable buffer is full, `writable.write()` returns `false`
3. `pipe` pauses the readable until `drain` fires
4. When buffer drains, `drain` fires → readable resumes

**Manual backpressure:**
```js
readable.on('data', (chunk) => {
  const canContinue = writable.write(chunk);
  if (!canContinue) {
    readable.pause(); // pause reading
    writable.once('drain', () => readable.resume());
  }
});
```

### 💻 Code Example
```js
const { Transform, pipeline } = require('stream');
const { promisify } = require('util');
const pipelineAsync = promisify(pipeline);

// Custom Transform stream
class CSVParser extends Transform {
  constructor() {
    super({ objectMode: true }); // output objects, not buffers
    this.buffer = '';
  }

  _transform(chunk, encoding, callback) {
    this.buffer += chunk.toString();
    const lines = this.buffer.split('\n');
    this.buffer = lines.pop(); // keep incomplete line

    lines.forEach(line => {
      if (line.trim()) {
        const [name, age, email] = line.split(',');
        this.push({ name, age: parseInt(age), email }); // push object
      }
    });
    callback();
  }

  _flush(callback) {
    if (this.buffer.trim()) {
      const [name, age, email] = this.buffer.split(',');
      this.push({ name, age: parseInt(age), email });
    }
    callback();
  }
}

// Process 1GB CSV without memory issues
async function processLargeCSV() {
  const parser = new CSVParser();
  const records = [];

  await pipelineAsync(
    fs.createReadStream('./users.csv'),
    parser,
    async function* (source) {
      for await (const record of source) {
        records.push(record); // or write to DB in batches
        yield record;
      }
    }
  );
  return records;
}
```

### ⚠️ Common Mistakes
- Collecting all chunks into an array before processing — defeats the purpose of streaming
- Using `objectMode: true` without understanding memory implications — objects aren't subject to highwater mark byte counting

### 🎯 Interview Tip
> "Backpressure is why Node.js can serve a 10GB file without loading it in memory — `pipe` manages the read/write rate automatically."

---

## Q3 — What is a Buffer in Node.js?

### ✅ Simple Explanation
A `Buffer` is a fixed-size chunk of memory allocated outside of V8's heap. It's used to work with binary data — file contents, network packets, crypto.

### 🧠 Deep Dive
- Buffers are instances of `Uint8Array`
- They're used because JavaScript strings aren't efficient for binary data
- `Buffer.alloc(n)` — safe, zero-filled
- `Buffer.allocUnsafe(n)` — faster, may contain old memory (security risk if exposed)
- `Buffer.from()` — create from string, array, or another buffer

**Encoding options:** `utf8`, `hex`, `base64`, `ascii`, `latin1`

### 💻 Code Example
```js
// Creating Buffers
const buf1 = Buffer.alloc(10);             // 10 bytes, zero-filled
const buf2 = Buffer.allocUnsafe(10);       // 10 bytes, uninitialized (FAST but risky)
const buf3 = Buffer.from('Hello World');   // from string (UTF-8 default)
const buf4 = Buffer.from([0x48, 0x65]);    // from byte array

// Reading/writing
console.log(buf3.toString('utf8'));        // 'Hello World'
console.log(buf3.toString('hex'));         // '48656c6c6f...'
console.log(buf3.toString('base64'));      // 'SGVsbG8gV29ybGQ='

// Buffer operations
const combined = Buffer.concat([buf3, buf4]);
console.log(buf3.length);                  // byte length, not char length

// Practical: encode JWT payload
const payload = { userId: 123 };
const encoded = Buffer.from(JSON.stringify(payload)).toString('base64');
const decoded = JSON.parse(Buffer.from(encoded, 'base64').toString());

// Safe use of allocUnsafe
const safeBuf = Buffer.allocUnsafe(16);
safeBuf.fill(0); // zero it out before use

// Stream chunks are Buffers
readStream.on('data', (chunk) => {
  console.log(Buffer.isBuffer(chunk)); // true
  console.log(chunk.toString('utf8')); // convert to string
});
```

### ⚠️ Common Mistakes
- Using `Buffer.allocUnsafe` and returning it to clients without zeroing — may expose old process memory
- Confusing `.length` (bytes) with string character count for multibyte characters
- Doing `new Buffer()` — deprecated and removed in Node 10+

### 🎯 Interview Tip
> "In production, always prefer `Buffer.alloc` for security. `Buffer.allocUnsafe` is only justified in hot paths where you immediately fill the buffer."

---

## Q4 — What is the difference between `readFile` vs `createReadStream`?

### ✅ Simple Explanation
- `fs.readFile()` — reads entire file into memory, then calls callback
- `fs.createReadStream()` — reads in chunks, emits `data` events, low memory

### 🧠 Deep Dive

| Feature | `readFile` | `createReadStream` |
|---|---|---|
| Memory usage | Entire file in RAM | 64KB default chunks |
| Speed (small files) | Faster (one syscall) | Slightly slower |
| Speed (large files) | Slower + memory risk | Much better |
| Streaming API | ❌ | ✅ |
| Use case | Config files, small data | Video, CSV, logs |

**Rule:** If file > a few MB, use streams.

### 💻 Code Example
```js
const fs = require('fs').promises;

// ✅ readFile — good for small config files
async function loadConfig() {
  const raw = await fs.readFile('./config.json', 'utf8');
  return JSON.parse(raw);
}

// ✅ createReadStream — for HTTP file serving
app.get('/download/:filename', (req, res) => {
  const filepath = path.join(__dirname, 'uploads', req.params.filename);
  const stat = fs.statSync(filepath);

  res.setHeader('Content-Type', 'application/octet-stream');
  res.setHeader('Content-Length', stat.size);

  const stream = fs.createReadStream(filepath);
  stream.pipe(res);
});

// Range requests (video streaming)
app.get('/video', (req, res) => {
  const filepath = './video.mp4';
  const stat = fs.statSync(filepath);
  const range = req.headers.range;

  if (range) {
    const [startStr, endStr] = range.replace('bytes=', '').split('-');
    const start = parseInt(startStr);
    const end = endStr ? parseInt(endStr) : stat.size - 1;
    const chunkSize = end - start + 1;

    res.writeHead(206, {
      'Content-Range': `bytes ${start}-${end}/${stat.size}`,
      'Content-Length': chunkSize,
      'Content-Type': 'video/mp4',
    });

    fs.createReadStream(filepath, { start, end }).pipe(res);
  }
});
```

### ⚠️ Common Mistakes
- Using `readFileSync` in route handlers — blocks event loop entirely
- Not handling stream errors before piping to response — results in incomplete responses

### 🎯 Interview Tip
> "For file downloads in a REST API, I always stream using `createReadStream`. For config loading at startup, `readFile` is fine since it runs before the server starts accepting requests."

---

## Q5 — How do you watch files for changes in Node.js?

### ✅ Simple Explanation
Use `fs.watch()` or `fs.watchFile()` to monitor file or directory changes. These are the building blocks for tools like `nodemon`.

### 🧠 Deep Dive
- `fs.watch()` — uses OS events (inotify/kqueue), fast, but has edge cases
- `fs.watchFile()` — uses polling, slower, but more reliable across network filesystems
- `chokidar` — popular npm library that wraps both with better cross-platform behavior

**`fs.watch` limitations:**
- Event callback may fire multiple times for a single change
- `rename` event covers both rename and delete
- Unreliable on network drives

### 💻 Code Example
```js
const fs = require('fs');
const path = require('path');

// Basic file watching
const watcher = fs.watch('./config.json', { encoding: 'utf8' }, (event, filename) => {
  if (event === 'change') {
    console.log(`${filename} was modified. Reloading...`);
    // reload config
  }
});

// Watch directory recursively
fs.watch('./src', { recursive: true }, (event, filename) => {
  console.log(`[${event}] ${filename}`);
});

// Debounce for editors that trigger multiple events
let timer;
fs.watch('./app.js', () => {
  clearTimeout(timer);
  timer = setTimeout(() => {
    console.log('File changed — restarting');
    // restart logic
  }, 100);
});

// Production recommendation: use chokidar
const chokidar = require('chokidar');
const watcher2 = chokidar.watch('./src', {
  ignored: /node_modules/,
  persistent: true
});
watcher2.on('change', path => console.log(`Changed: ${path}`));
watcher2.on('add', path => console.log(`Added: ${path}`));
watcher2.on('unlink', path => console.log(`Deleted: ${path}`));

// Don't forget to close watchers
process.on('SIGINT', () => watcher.close());
```

### ⚠️ Common Mistakes
- Not debouncing `fs.watch` — editors save in multiple steps, triggering multiple events
- Using `fs.watchFile` in production (polling) when `fs.watch` (OS events) is more efficient

### 🎯 Interview Tip
> "`fs.watch` is event-driven and fast. `chokidar` is preferred in production tools because it handles edge cases across OSes and supports recursive watching reliably."

---

## ⚡ Quick Revision Summary

- Streams process data in chunks → low memory footprint for large files
- 4 stream types: Readable, Writable, Duplex, Transform
- Backpressure: `write()` returns `false` → pause readable → resume on `drain`
- `pipeline()` > `.pipe()` — proper error propagation and cleanup
- `Buffer` = raw binary data outside V8 heap; `Buffer.alloc` is safe, `Buffer.allocUnsafe` is fast
- `readFile` = entire file in memory; `createReadStream` = chunk-by-chunk
- `fs.watch` = OS events (fast); `fs.watchFile` = polling (slow); use `chokidar` in production

---

## 🏆 Top 5 Must Revise Before Interview

1. **Streams vs readFile** — when to use which (memory implications)
2. **Backpressure mechanism** — how pipe handles slow consumers
3. **pipeline vs pipe** — error handling difference
4. **Buffer.alloc vs allocUnsafe** — security implications
5. **Custom Transform stream** — shows deep understanding

---

## 🎤 Real Interview Questions

- *"How would you serve a 2GB file download from a Node.js server without crashing it?"*
- *"What is backpressure and how does Node.js handle it?"*
- *"What's the difference between `pipe` and `pipeline`?"*
- *"When would you use a Transform stream? Give a real example."*
