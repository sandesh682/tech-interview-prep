# 🟣 05 — Streams and File System

---

📍 **Progress Marker**
- Last Revised: ___________
- Revision Count: 0
- Difficulty Level: 🟣 Medium–Hard

---
# Q1 —What is Buffers in Node.js?

---

# 📌 What is a Buffer?

- A **Buffer** is a fixed-size memory block used to store **raw binary data (bytes)**.
    
- It represents data in its **lowest-level format**.
    

👉 One-line:

> Buffer = raw binary data stored in memory

---

# ❓ Why Buffers Exist

- Node.js deals heavily with:
    
    - File systems (images, videos, PDFs)
        
    - Network requests (HTTP, TCP)
        
    - Streams
        

👉 These are **binary operations**, not just text.

> Node does NOT assume data is string → uses Buffer by default

---

# 🔢 Example

```js
const buffer = Buffer.from("Hello");

console.log(buffer);           // <Buffer 48 65 6c 6c 6f>
console.log(buffer.toString()); // Hello
```

---

# 🔍 Internal Representation

- Stored as **hexadecimal bytes**
    
- Each byte = 8 bits
    

```text
48 65 6c 6c 6f → "Hello"
```

---

# ⚙️ Creating Buffers

## 1. From Data

```js
Buffer.from("Hello");
```

---

## 2. Safe Allocation

```js
Buffer.alloc(size);
```

- Allocates memory
    
- Fills with `0`
    
- Safe
    

---

## 3. Unsafe Allocation

```js
Buffer.allocUnsafe(size);
```

- Allocates memory
    
- Does NOT initialize
    
- Faster but risky
    

---

# ⚖️ alloc vs allocUnsafe

|Feature|Buffer.alloc|Buffer.allocUnsafe|
|---|---|---|
|Initialization|Zero-filled|Not initialized|
|Safety|Safe|Unsafe|
|Speed|Slower|Faster|
|Use case|General use|High-performance|

---

# 🧠 Why allocUnsafe is Faster

- Skips memory initialization step
    
- Directly returns allocated memory
    

```text
alloc        → allocate + clean
allocUnsafe  → allocate only
```

---

# ⚠️ Risk of allocUnsafe

- May contain:
    
    - Old memory data
        
    - Sensitive info
        
- Must overwrite before use
    

```js
const buf = Buffer.allocUnsafe(10);
buf.fill(0); // make safe
```

---

# 🧩 Core Properties

## Fixed Size

```js
const buf = Buffer.alloc(10);
```

---

## Indexed Access

```js
buf[0] = 65; // 'A'
```

---

## Length

```js
buf.length;
```

---

## Encoding Support

```js
buf.toString("utf-8");
buf.toString("hex");
buf.toString("base64");
```

---

# 🔄 Buffer Operations

## Slice (Shared Memory ⚠️)

```js
const part = buf.slice(0, 5);
```

- Does NOT copy
    
- References same memory
    

---

## Copy

```js
buf1.copy(buf2);
```

---

## Concat

```js
Buffer.concat([buf1, buf2]);
```

---

# 🔌 Buffers in Streams (VERY IMPORTANT)

- Streams process data in chunks
    
- Each chunk is a **Buffer**
    

```js
stream.on("data", (chunk) => {
  // chunk is Buffer
});
```

---

# 📂 Buffers in File System (Low-Level)

```js
const buffer = Buffer.alloc(1024);

fs.read(fd, buffer, 0, 1024, 0, (err, bytesRead) => {
  console.log(buffer.toString("utf-8", 0, bytesRead));
});
```

👉 Buffer acts as **temporary memory to hold file data**

---

# 🌐 Buffers in Networking

```js
req.on("data", (chunk) => {
  // Buffer from network
});
```

👉 Used in:

- HTTP requests
    
- WebSockets
    
- TCP servers
    

---

# ⚡ Direct vs Indirect Usage

## 🔹 Direct Usage (Rare)

- Binary protocols
    
- Custom file readers
    
- Performance-critical systems
    

---

## 🔹 Indirect Usage (Common)

- Streams (`fs.createReadStream`)
    
- HTTP requests/responses
    
- File uploads/downloads
    

👉 Most Node apps use Buffers **implicitly**

---

# 🧠 Mental Model

```text
File / Network
      ↓
   Stream
      ↓
  Buffer (chunks)
      ↓
Convert (string/json/etc)
      ↓
   Use data
```

---

# ⚠️ Important Concepts

## Node is Binary-First

- Everything comes as Buffer first
    

---

## Partial Data Handling

- Buffer may not be fully filled
    
- Always use `bytesRead`
    

---

## Encoding Awareness

- Same data → different encodings
    

---

## Memory Efficiency

- Buffers avoid loading full data
    
- Enable chunk-based processing
    

---

# 🔥 Interview Points

- Buffer stores **raw binary data**
    
- Default data format in Node I/O
    
- Fixed-size memory allocation
    
- Used in streams and networking
    
- `alloc` vs `allocUnsafe` trade-off (safety vs performance)
    
- Slice shares memory (important!)
    

---

# 🚀 Real-World Use Cases

- File upload/download systems
    
- Video/audio streaming
    
- API request handling
    
- TCP/socket communication
    
- Binary data processing
    

---

# ✅ Summary

- Buffer = raw bytes in memory
    
- Core to Node.js I/O
    
- Used everywhere (directly or indirectly)
    
- Enables efficient, scalable data processing
    
- `alloc` = safe, `allocUnsafe` = fast
    

---
## Q2 — What are Streams in Node.js and why are they important?

---

# 📌 What is a Stream?

- A **stream** is a way to handle data **piece by piece (chunks)** instead of loading everything at once.
    
- Used for **efficient I/O operations** (files, network, etc.)
    

👉 One-line:

> Stream = continuous flow of data in chunks

---

# 🔢 Types of Streams in Node.js

There are **4 main types of streams**:

---

# 🔹 1. Readable Stream

👉 Used to **read data from a source**

### Examples:

- Reading a file
    
- Incoming HTTP request
    
- Database read
    

```js
fs.createReadStream("file.txt");
```

---

### 🧠 Key Idea:

> Data flows **from source → your application**

---

# 🔹 2. Writable Stream

👉 Used to **write data to a destination**

### Examples:

- Writing to a file
    
- Sending HTTP response
    
- Writing logs
    

```js
fs.createWriteStream("file.txt");
```

---

### 🧠 Key Idea:

> Data flows **from your application → destination**

---

# 🔹 3. Duplex Stream

👉 Can **read AND write**

### Examples:

- TCP sockets
    
- WebSockets
    

---

### 🧠 Key Idea:

> Two-way data flow

```text
Read ⇄ Write
```

---

# 🔹 4. Transform Stream 🔥

👉 A special type of duplex stream that **modifies data while passing**

### Examples:

- Compression (gzip)
    
- Encryption
    
- Data formatting
    

---

### 🧠 Key Idea:

```text
Input → Transform → Output
```

---

# ⚖️ Quick Comparison

|Type|Read|Write|Modify Data|
|---|---|---|---|
|Readable|✅|❌|❌|
|Writable|❌|✅|❌|
|Duplex|✅|✅|❌|
|Transform|✅|✅|✅|

---

# 🔄 Data Flow Overview

```text
Readable → Transform → Writable
```

---

# 🧠 Mental Model

```text
Source (Readable)
        ↓
   Processing (Transform)
        ↓
Destination (Writable)
```

---

# ⚡ Real-World Example

👉 File Copy

```text
Read file → (optional transform) → Write file
```

---

# 🔥 Important Concepts (Will Cover Later)

- Flowing vs Paused mode
    
- Backpressure
    
- pipe()
    
- highWaterMark
    

---

# 🚀 Usage in Real Systems

- File uploads/downloads
    
- Video/audio streaming
    
- API request handling
    
- Data pipelines
    
- Real-time communication
    

---

# 🎯 Interview Points

- Streams process data in chunks
    
- Improve memory efficiency
    
- 4 types: Readable, Writable, Duplex, Transform
    
- Core to Node.js I/O system
    

---

# ✅ Summary

- Streams = chunk-based data handling
    
- Enable scalable systems
    
- Each type serves a specific role in data flow
    
- Often combined together (pipeline)
    

---
# Q3 — What is Readable Streams in Node.js?

---

# 📌 What is a Readable Stream?

- A **Readable Stream** is a stream from which data can be **read chunk by chunk**.
    
- It represents a **data source**.
    

👉 One-line:

> Readable stream = source of data flowing in chunks

---

# 🔢 Common Examples

- File reading → `fs.createReadStream()`
    
- HTTP request → `req`
    
- Network sockets
    
- Database streams
    

---

# 🧩 Basic Example

```js
const fs = require("fs");

const stream = fs.createReadStream("file.txt");

stream.on("data", (chunk) => {
  console.log(chunk.toString());
});
```

---

# 🧠 Internal Working (IMPORTANT)

```text
File / Source
      ↓
Internal Buffer (~64KB)
      ↓
Readable Stream
      ↓
"data" events (chunks)
      ↓
Your code
```

---

# ⚙️ How It Works Step-by-Step

1. Open source (file/socket)
    
2. Allocate buffer (default ~64KB)
    
3. Read chunk into buffer
    
4. Emit `data` event
    
5. Repeat until done
    
6. Emit `end` event
    

---

# 📦 Important Events

## 1. data

- Fired when chunk is available
    

```js
stream.on("data", chunk => {});
```

---

## 2. end

- Fired when no more data
    

```js
stream.on("end", () => {});
```

---

## 3. error

- Fired on failure
    

```js
stream.on("error", err => {});
```

---

## ⚠️ Flowing vs Paused Mode (VERY IMPORTANT)

---

## 🔹 Flowing Mode

👉 Stream automatically pushes data

Triggered by:

```js
stream.on("data", ...)
```

### Behavior:

```text
Stream → continuously emits data
```

---

## 🔹 Paused Mode

👉 You manually read data

```js
stream.on("readable", () => {
  let chunk;
  while ((chunk = stream.read()) !== null) {
    console.log(chunk.toString());
  }
});
```

---

## 🧠 Key Difference

|Mode|Control Flow|
|---|---|
|Flowing|Stream|
|Paused|Developer|

---

# 🔄 Switching Modes

stream.pause();
stream.resume();

```js
stream.on("data", (chunk) => {
  console.log(chunk.toString());
  stream.pause();

  setTimeout(() => {
    stream.resume();
  }, 1000);
});
```

---

# ⚡ highWaterMark (Chunk Size Control)

```js
fs.createReadStream("file.txt", {
  highWaterMark: 64 * 1024
});
```

- Default ≈ 64KB
    
- Controls chunk size
    

---

# ⚠️ Important Concepts

## Buffer-Based

- Each chunk is a **Buffer**
    

```js
stream.on("data", chunk => {
  // chunk is Buffer
});
```

---

## Lazy Execution

- Stream does NOT start automatically
    
- Starts when:
    
    - `data` listener added
        
    - `.pipe()` called
        
    - `.resume()` called
        

---

## Partial Reads

- Chunk ≠ full file
    
- File is split into multiple chunks
    

---

# 🔥 Real-World Usage

- Reading large files (GB/TB scale)
    
- Video/audio streaming
    
- File uploads/downloads
    
- Processing logs
    

---

# 🧠 Mental Model

```text
Source
  ↓
Readable Stream
  ↓
Chunks (Buffer)
  ↓
Your logic
```

---

# ⚡ Example: Stream a File

```js
const fs = require("fs");

fs.createReadStream("file.txt")
  .on("data", chunk => console.log(chunk.toString()))
  .on("end", () => console.log("Done"));
```

---

# 🔥 Interview Points

- Readable streams read data in chunks
    
- Event-driven (`data`, `end`, `error`)
    
- Flowing vs paused modes
    
- Uses internal buffer (~64KB default)
    
- Efficient for large data handling
    

---

# ✅ Summary

- Readable stream = data source
    
- Works in chunks (Buffer)
    
- Supports flowing & paused modes
    
- Core part of Node.js I/O
    

---

## Q2 — What is Writable Streams and Backpressure in Node.js?

---

# 📌 What is a Writable Stream?

- A **Writable Stream** is a destination where data is **written chunk by chunk**.
    
- Opposite of Readable stream.
    

👉 One-line:

> Writable stream = destination that consumes data in chunks

---

# 🔢 Common Examples

- Writing to file → `fs.createWriteStream()`
    
- HTTP response → `res`
    
- Logging systems
    
- Network sockets
    

---

# 🧩 Basic Example

```js
const fs = require("fs");

const writeStream = fs.createWriteStream("output.txt");

writeStream.write("Hello ");
writeStream.write("World\n");

writeStream.end("Done!");
```

---

# 🧠 Internal Working

```text
Your Code → Writable Stream → Internal Buffer → Destination (file/socket)
```

---

# ⚡ Important: `.write()` is Buffered

```js
writeStream.write(chunk);
```

👉 Data is:

- First written to **internal buffer**
    
- Then flushed to destination
    

👉 NOT immediately written to disk/network

---

# 🚨 Backpressure (MOST IMPORTANT)

## 📌 What is Backpressure?

> When data is written faster than it can be processed

---

## 🔥 Example

```text
Producer (write calls) → fast
Consumer (disk/network) → slow

💥 Buffer fills → backpressure
```

---

# ⚡ `.write()` Return Value

```js
const canWrite = writeStream.write(chunk);
```

|Value|Meaning|
|---|---|
|true|Buffer has space|
|false|Buffer is full 🚨|

---

# 🚨 Handling Backpressure

## ❗ When `.write()` returns false:

👉 STOP writing

```js
if (!canWrite) {
  writeStream.once("drain", writeMore);
}
```

---

# ⚡ `drain` Event

- Fired when buffer is **empty again**
    
- Signal to **resume writing**
    

```js
writeStream.once("drain", () => {
  // resume writing
});
```

---

# 🔄 Flow Control Pattern

```text
write() → returns false
        ↓
STOP writing
        ↓
wait for "drain"
        ↓
resume writing
```

---

# ⚙️ highWaterMark

```js
fs.createWriteStream("file.txt", {
  highWaterMark: 16 * 1024 // 16KB
});
```

- Defines buffer size limit
    
- When exceeded → `.write()` returns false
    

---

# 🧪 Proper Backpressure Handling Pattern

```js
let i = 0;

function writeData() {
  let canWrite = true;

  while (i < 100000 && canWrite) {
    canWrite = writeStream.write(`Line ${i}\n`);
    i++;
  }

  if (i < 100000) {
    writeStream.once("drain", writeData);
  } else {
    writeStream.end();
  }
}

writeData();
```

---

# 🔌 Manual pipe (Manual Flow)

```js
readStream.on("data", (chunk) => {
  const canWrite = writeStream.write(chunk);

  if (!canWrite) {
    readStream.pause();

    writeStream.once("drain", () => {
      readStream.resume();
    });
  }
});
```

---

# 🚀 pipe() (IMPORTANT)

```js
readStream.pipe(writeStream);
```

👉 Automatically:

- Handles backpressure
    
- Pauses/resumes
    
- Manages flow
    

---

# 🧠 Mental Model

```text
Readable (producer) → Writable (consumer)

If producer > consumer → backpressure
```

---

# ⚠️ Common Mistakes

❌ Ignoring `.write()` return value  
❌ Writing continuously without control  
❌ Not handling `drain`  
❌ Assuming write is instant

---

# 🔥 Real-World Usage

- File writing systems
    
- Logging pipelines
    
- Data processing pipelines
    
- Upload services
    
- Streaming systems
    

---

# 🎯 Interview Points

- Writable streams consume data in chunks
    
- `.write()` is buffered
    
- Returns boolean for backpressure control
    
- `false` → wait for `drain`
    
- `highWaterMark` controls buffer size
    
- `pipe()` handles backpressure automatically
    

---

# ✅ Summary

- Writable stream = data destination
    
- Uses internal buffer
    
- Backpressure prevents overload
    
- `drain` enables controlled flow
    
- Critical for scalable systems
    

---

# Q3 — What is Duplex Stream in Node.js?

---

# 📌 What is a Duplex Stream?

- A **Duplex Stream** is a stream that is both:
    
    - **Readable** (can read data)
        
    - **Writable** (can write data)
        

👉 One-line:

> Duplex = Read + Write in a single stream

---

# 🔢 Conceptual View

```text
Readable  ⇄  Writable
```

👉 Two-way data flow

---

# 🧩 Examples

- TCP sockets → `net.Socket`
    
- WebSockets
    
- Some file/network streams
    

---

# 🧠 Key Characteristics

- Supports **reading and writing independently**
    
- Has separate internal buffers:
    
    - Read buffer
        
    - Write buffer
        
- Can perform **bi-directional communication**
    

---

# ⚙️ Basic Example (Conceptual)

```js
const { Duplex } = require("stream");

const duplex = new Duplex({
  read(size) {
    this.push("Hello");
    this.push(null);
  },
  write(chunk, encoding, callback) {
    console.log("Writing:", chunk.toString());
    callback();
  }
});

duplex.on("data", (chunk) => {
  console.log("Read:", chunk.toString());
});

duplex.write("World");
```

---

# 🧠 How It Works

```text
Write → internal write buffer → processed
Read  → internal read buffer  → emitted
```

👉 Read and write sides are **independent**

---

# ⚠️ Important Concept

👉 Duplex streams do NOT guarantee:

- Read speed = Write speed
    

👉 Each side has:

- Separate flow control
    
- Separate buffering
    

---

# 🔄 Data Flow

```text
Input (write)
     ↓
[Duplex Stream]
     ↓
Output (read)
```

---

# 🧠 Duplex vs Other Streams

|Type|Read|Write|Modify Data|
|---|---|---|---|
|Readable|✅|❌|❌|
|Writable|❌|✅|❌|
|Duplex|✅|✅|❌|
|Transform|✅|✅|✅|

---

# 🔥 Duplex vs Transform (IMPORTANT)

👉 Transform is a **special type of Duplex**

|Feature|Duplex|Transform|
|---|---|---|
|Read/Write|Independent|Linked|
|Data Modify|❌|✅|

---

## 🧠 Explanation

- Duplex:
    
    - Input and output are separate
        
- Transform:
    
    - Output depends on input
        

---

# ⚡ Real-World Usage

- Network communication (client ↔ server)
    
- Chat applications
    
- WebSocket connections
    
- Streaming pipelines
    

---

# 🔌 Example: Socket (Real Duplex)

```js
socket.write("Hello");
socket.on("data", (data) => {
  console.log(data.toString());
});
```

👉 Same object:

- Sends data
    
- Receives data
    

---

# 🧠 Mental Model

```text
Two pipes in one system:

Write pipe → outgoing data  
Read pipe  → incoming data  
```

---

# ⚠️ Common Misconceptions

❌ Duplex automatically transforms data  
✔ It only enables two-way flow

❌ Read and write are synchronized  
✔ They are independent

---

# 🔥 Interview Points

- Duplex streams support both read and write
    
- Used in bidirectional communication (e.g., sockets)
    
- Read and write sides are independent
    
- Transform streams extend Duplex
    

---

# ✅ Summary

- Duplex = Read + Write in one stream
    
- Enables two-way data flow
    
- Used mainly in networking
    
- Less commonly implemented manually
    
- Foundation for Transform streams
    

---
# Q4 — What are Transform Streams in Node.js?

---

# 📌 What is a Transform Stream?

- A **Transform Stream** is a special type of **Duplex stream** that:
    
    - Reads data (input)
        
    - Modifies it
        
    - Outputs transformed data
        

👉 One-line:

> Transform = Read + Modify + Write

---

# 🔄 Data Flow

```text
Input → Transform → Output
```

---

# 🧩 Key Characteristics

- Both **readable** and **writable**
    
- Output is **derived from input**
    
- Processes data **chunk by chunk (Buffer)**
    
- Used in streaming pipelines
    

---

# ⚙️ Core Method: `transform()`

```js
transform(chunk, encoding, callback)
```

---

## 🔍 Parameters

### 1. `chunk`

- Incoming data (Buffer)
    

---

### 2. `encoding`

- Encoding type (usually ignored)
    

---

### 3. `callback`

- Must be called after processing
    

```js
callback(null, transformedData);
```

---

# ⚠️ Important Rules

- Always call `callback()` ❗
    
- Do NOT assume full data (chunk-based processing)
    
- Keep logic non-blocking
    

---

# 🧪 Basic Example (Uppercase)

```js
const { Transform } = require("stream");

const upperCaseTransform = new Transform({
  transform(chunk, encoding, callback) {
    const result = chunk.toString().toUpperCase();
    callback(null, result);
  }
});
```

---

# 🔌 Usage with Streams

```js
fs.createReadStream("input.txt")
  .pipe(upperCaseTransform)
  .pipe(fs.createWriteStream("output.txt"));
```

---

# 🧠 Internal Working

```text
Readable → transform() → Writable
```

Steps:

1. Receive chunk
    
2. Process it
    
3. Push transformed output
    

---

# 🔥 Transform vs Duplex

|Feature|Duplex|Transform|
|---|---|---|
|Read/Write|Independent|Linked|
|Modify Data|❌|✅|

---

# ⚡ Common Use Cases

- Data formatting (uppercase/lowercase)
    
- Compression (gzip)
    
- Encryption/decryption
    
- Parsing (CSV → JSON)
    
- Filtering logs/streams
    

---

# 🧠 Mental Model

```text
Readable → Transform → Writable
          ↑
     modifies data
```

---

# ⚠️ Common Mistakes

- Forgetting `callback()` → stream hangs ❌
    
- Blocking code inside transform ❌
    
- Assuming entire file is available ❌
    

---

# 🔥 Interview Points

- Transform streams modify data in flow
    
- Special type of Duplex stream
    
- Uses `transform()` method
    
- Works with `pipe()` for chaining
    
- Efficient for large data processing
    

---

# ✅ Summary

- Transform = read + modify + write
    
- Chunk-based processing
    
- Core for streaming pipelines
    
- Memory efficient for large data
    

---
# Q6 — pipe() vs pipeline() in Node.js

---

# 📌 Overview

- Both `pipe()` and `pipeline()` are used to **connect streams**
    
- They enable **data flow from source → destination**
    

👉 One-line:

```text
pipe() = basic connection
pipeline() = safe, production-ready connection
```

---

# 🔹 pipe()

## 📌 What is pipe()?

- A method used to **connect streams**
    
- Transfers data automatically
    

```js
readable.pipe(writable);
```

---

## 🔄 Flow

```text
Readable → Transform → Writable
```

---

## ✅ Features

- Simple and easy to use
    
- Handles **backpressure automatically**
    
- Good for small or quick tasks
    

---

## ⚠️ Limitations

- ❌ No automatic error handling
    
- ❌ Streams may not close properly on failure
    
- ❌ Requires manual cleanup
    

---

## 🧪 Example

```js
readStream
  .pipe(transformStream)
  .pipe(writeStream);
```

---

## ⚠️ Manual Error Handling Required

```js
readStream.on("error", handleError);
transformStream.on("error", handleError);
writeStream.on("error", handleError);
```

---

# 🔹 pipeline()

## 📌 What is pipeline()?

- A utility function to **connect streams safely**
    

```js
const { pipeline } = require("stream");
```

---

## 🧪 Example

```js
pipeline(
  readStream,
  transformStream,
  writeStream,
  (err) => {
    if (err) console.error("Failed:", err);
    else console.log("Success");
  }
);
```

---

## ✅ Features

- Automatic **error handling**
    
- Proper **stream cleanup**
    
- Prevents **memory leaks**
    
- Callback when completed
    

---

# ⚡ Async Version

```js
const { pipeline } = require("stream/promises");

await pipeline(
  readStream,
  transformStream,
  writeStream
);
```

---

# ⚖️ Comparison

|Feature|pipe()|pipeline()|
|---|---|---|
|Connect streams|✅|✅|
|Backpressure|✅|✅|
|Error handling|❌ manual|✅ automatic|
|Cleanup|❌ manual|✅ automatic|
|Production safe|❌|✅|

---

# 🧠 Internal Difference

```text
pipe()     → You manage lifecycle
pipeline() → Node manages lifecycle
```

---

# 🎯 When to Use

## ✅ pipe()

- Small scripts
    
- Simple data flow
    
- Learning/demo code
    

---

## 🔥 pipeline()

- Production systems
    
- File uploads/downloads
    
- APIs handling large data
    
- Critical backend logic
    

---

# 🧠 Mental Model

```text
pipe()     = basic pipe connection
pipeline() = managed pipeline with safety
```

---

# 🔥 Interview Points

- `pipe()` connects streams but lacks error handling
    
- `pipeline()` is safer and production-ready
    
- Both support backpressure
    
- Prefer `pipeline()` in real-world systems
    

---

# ✅ Summary

- Both connect streams
    
- `pipe()` is simple but limited
    
- `pipeline()` adds safety, error handling, and cleanup
    
- Use `pipeline()` for production
    

---
# Q5 — Node.js `fs` Module — Essential Notes

---

# 📌 What is `fs` Module?

- Built-in Node.js module to interact with the **file system**
    
- Used for:
    
    - Reading files 📖
        
    - Writing files ✍️
        
    - Managing directories 📁
        
    - File metadata 🔍
        

👉 One-line:

> `fs` = Node.js interface to the operating system’s file system

---

# ⚙️ Preferred Usage

```js
const fs = require("fs/promises");
```

👉 Use **Promise-based API (async/await)**  
❌ Avoid sync methods in production (blocking)

---

# 🔥 MUST-KNOW METHODS (CORE SET)

---

# 📖 1. Read File

```js
await fs.readFile("file.txt", "utf-8");
```

👉 Loads entire file into memory  
⚠️ Avoid for large files

### ❌ Sync Alternative (Avoid)

```js
fs.readFileSync("file.txt", "utf-8");
```

- Blocks event loop
    
- Not suitable for servers
    

---

# ✍️ 2. Write File

```js
await fs.writeFile("file.txt", "Hello World");
```

👉 Overwrites file

### ❌ Sync Alternative (Avoid)

```js
fs.writeFileSync("file.txt", "Hello World");
```

- Blocking operation
    

---

# ➕ 3. Append File

```js
await fs.appendFile("file.txt", "New line\n");
```

👉 Adds data to end

### ❌ Sync Alternative (Avoid)

```js
fs.appendFileSync("file.txt", "New line\n");
```

---

# ❌ 4. Delete File

```js
await fs.unlink("file.txt");
```

### ❌ Sync Alternative (Avoid)

```js
fs.unlinkSync("file.txt");
```

---

# 📁 5. Directory Operations

## Create directory

```js
await fs.mkdir("folder");
```

### ❌ Sync Alternative (Avoid)

```js
fs.mkdirSync("folder");
```

---

## Read directory

```js
const files = await fs.readdir(".");
```

### ❌ Sync Alternative (Avoid)

```js
fs.readdirSync(".");
```

---

## Remove directory

```js
await fs.rmdir("folder");
```

### ❌ Sync Alternative (Avoid)

```js
fs.rmdirSync("folder");
```

---

# 🔍 6. File Info (VERY IMPORTANT)

```js
const stats = await fs.stat("file.txt");
```

```js
stats.isFile();
stats.isDirectory();
```

### ❌ Sync Alternative (Avoid)

```js
const stats = fs.statSync("file.txt");
```

---

# 🔄 7. Rename / Move

```js
await fs.rename("old.txt", "new.txt");
```

### ❌ Sync Alternative (Avoid)

```js
fs.renameSync("old.txt", "new.txt");
```

---

# 📦 8. Copy File

```js
await fs.copyFile("a.txt", "b.txt");
```

### ❌ Sync Alternative (Avoid)

```js
fs.copyFileSync("a.txt", "b.txt");
```

---

# 🔥 9. Streams (Already Covered ✅)

```js
fs.createReadStream("file.txt");
fs.createWriteStream("file.txt");
```

👉 Used for large files, streaming, backpressure

---

# ⚠️ When to Use What

|Task|Method|
|---|---|
|Small file read|readFile|
|Large file read|createReadStream|
|Write file|writeFile|
|Continuous writing|createWriteStream|
|File metadata|stat|

---

# ⚠️ Best Practices

- Prefer **async (Promise-based)** methods
    
- Avoid sync methods in APIs/servers
    
- Use streams for large data
    
- Always handle errors (try/catch)
    

---

# 🔥 Interview Points

- Sync vs Async fs
    
- Blocking vs non-blocking
    
- readFile vs streams
    
- When to use each method
    

---

# 🧠 Mental Model

```text
Node.js → fs module → OS file system
```

---

# ✅ Summary

- `fs` is core for file operations
    
- Use async (`fs/promises`) methods
    
- Avoid sync methods in production
    
- Streams for large files (already covered)
    

---
## Q6 — What is the difference between `readFile` vs `createReadStream`?

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

### ⚠️ Common Mistakes
- Using `readFileSync` in route handlers — blocks event loop entirely
- Not handling stream errors before piping to response — results in incomplete responses

### 🎯 Interview Tip
> "For file downloads in a REST API, I always stream using `createReadStream`. For config loading at startup, `readFile` is fine since it runs before the server starts accepting requests."

---

## Q7 — How do you watch files for changes in Node.js?

---

# 📌 What is File Watching?

- Monitoring files/directories for changes (edit, delete, rename)
    
- Used in:
    
    - Dev servers (auto reload 🔄)
        
    - Build tools
        
    - Log monitoring
        
    - Config hot-reload
        

---

# 🔹 1. `fs.watch()` (Built-in)

## 🧪 Example

```js
const fs = require("fs");

fs.watch("file.txt", (eventType, filename) => {
  console.log(eventType, filename);
});
```

---

## ⚡ Events

|Event|Meaning|
|---|---|
|change|File content modified|
|rename|File renamed or deleted|

---

## ⚠️ Limitations

- Not reliable across OS ❌
    
- May miss events ❌
    
- Filename can be `null` ❌
    
- Recursive watching is limited
    

---

# 🔹 2. `fs.watchFile()` (Polling)

```js
fs.watchFile("file.txt", (curr, prev) => {
  console.log("File changed");
});
```

---

## ⚠️ Drawbacks

- Uses polling (checks repeatedly)
    
- Slower ❌
    
- Higher CPU usage ❌
    

---

# 🔥 3. `chokidar` (Production Standard)

👉 chokidar

---

# 📌 What is chokidar?

- A **high-level file watching library**
    
- Built on top of:
    
    - `fs.watch`
        
    - `fs.watchFile`
        
- Fixes cross-platform issues
    

---

# 🧪 Basic Example

```js
const chokidar = require("chokidar");

const watcher = chokidar.watch("file.txt");

watcher.on("change", (path) => {
  console.log(`File changed: ${path}`);
});
```

---

# 🔥 Common Events

```js
watcher
  .on("add", path => console.log("File added"))
  .on("change", path => console.log("File changed"))
  .on("unlink", path => console.log("File deleted"));
```

---

# ⚡ Watch Directory

```js
chokidar.watch("./src").on("all", (event, path) => {
  console.log(event, path);
});
```

---

# 🧠 Why chokidar is Better

- ✅ Cross-platform reliable
    
- ✅ Handles edge cases
    
- ✅ Supports recursive watching
    
- ✅ Debounces rapid changes
    
- ✅ Better event consistency
    

---

# ⚙️ Useful Options

```js
chokidar.watch("src", {
  ignored: /node_modules/,
  persistent: true,
  ignoreInitial: true
});
```

---

# 🚀 Real-World Usage

- Webpack
    
- Vite
    
- Next.js
    
- Nodemon
    

---

# 🔥 Which One Restarts React Server?

👉 React dev servers (like Vite / Webpack / CRA) use:

> 🔥 **chokidar internally**

---

## 🧠 Flow

```text
File change → chokidar detects → rebuild → browser reload
```

---

# 🔄 Example: Auto Restart Node Server

```js
const chokidar = require("chokidar");
const { spawn } = require("child_process");

let server;

function startServer() {
  if (server) server.kill();

  server = spawn("node", ["app.js"], { stdio: "inherit" });
}

chokidar.watch("./").on("change", () => {
  console.log("Restarting server...");
  startServer();
});

startServer();
```

---

# ⚖️ Comparison

|Feature|fs.watch|fs.watchFile|chokidar|
|---|---|---|---|
|Mechanism|Event-based|Polling|Hybrid|
|Performance|Fast|Slow|Optimized|
|Reliability|Medium|High|Very High|
|Cross-platform|Poor|Good|Excellent|
|Production|❌|❌|✅|

---

# 🎯 Interview Points

- `fs.watch()` is native but unreliable
    
- `fs.watchFile()` uses polling (slow)
    
- `chokidar` is industry standard
    
- Used in dev tools like React, Vite, Webpack
    

---

# 🧠 Mental Model

```text
fs.watch     → low-level watcher
chokidar     → production-ready watcher
```

---

# ✅ Summary

- Use `fs.watch` for simple cases
    
- Avoid `fs.watchFile` unless needed
    
- Use `chokidar` for real-world apps
    
- React dev servers rely on chokidar
    

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
