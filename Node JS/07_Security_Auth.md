# ⚫ 07 — Security & Authentication

---

📍 **Progress Marker**
- Last Revised: ___________
- Revision Count: 0
- Difficulty Level: ⚫ Hard

---

## Q1 — How does JWT work? Explain the full flow.

### ✅ Simple Explanation
JWT (JSON Web Token) is a stateless authentication mechanism. The server signs a token with a secret; the client stores it and sends it with each request; the server verifies the signature without hitting the database.

### 🧠 Deep Dive
**JWT Structure** (3 parts, base64url-encoded, separated by `.`):
```
Header.Payload.Signature
eyJ...  .eyJ...  .xyz...
```
- **Header**: `{ "alg": "HS256", "typ": "JWT" }`
- **Payload**: `{ "sub": "userId", "iat": 1234, "exp": 1234 }`
- **Signature**: `HMACSHA256(base64(header) + "." + base64(payload), secret)`

**Access + Refresh token pattern:**
- Access token: short-lived (15 min), stored in memory
- Refresh token: long-lived (7 days), stored in httpOnly cookie
- On access token expiry, use refresh token to get new access token

**Why not store JWT in localStorage?** XSS attack can steal it. Use memory + httpOnly cookies.

### 💻 Code Example
```js
const jwt = require('jsonwebtoken');

// Sign tokens
function generateTokens(userId) {
  const accessToken = jwt.sign(
    { sub: userId, type: 'access' },
    process.env.JWT_ACCESS_SECRET,
    { expiresIn: '15m' }
  );

  const refreshToken = jwt.sign(
    { sub: userId, type: 'refresh' },
    process.env.JWT_REFRESH_SECRET,
    { expiresIn: '7d' }
  );

  return { accessToken, refreshToken };
}

// Auth middleware
function authenticateToken(req, res, next) {
  const authHeader = req.headers.authorization;
  const token = authHeader?.startsWith('Bearer ') && authHeader.split(' ')[1];

  if (!token) return res.status(401).json({ error: 'No token' });

  try {
    const payload = jwt.verify(token, process.env.JWT_ACCESS_SECRET);
    if (payload.type !== 'access') throw new Error('Wrong token type');
    req.user = payload;
    next();
  } catch (err) {
    if (err.name === 'TokenExpiredError') {
      return res.status(401).json({ error: 'Token expired', code: 'TOKEN_EXPIRED' });
    }
    res.status(401).json({ error: 'Invalid token' });
  }
}

// Refresh token endpoint
app.post('/auth/refresh', async (req, res) => {
  const refreshToken = req.cookies.refreshToken;
  if (!refreshToken) return res.status(401).json({ error: 'No refresh token' });

  try {
    const payload = jwt.verify(refreshToken, process.env.JWT_REFRESH_SECRET);
    // Optional: check if token is in a revocation list (Redis)
    const { accessToken, refreshToken: newRefresh } = generateTokens(payload.sub);

    res.cookie('refreshToken', newRefresh, {
      httpOnly: true,
      secure: process.env.NODE_ENV === 'production',
      sameSite: 'strict',
      maxAge: 7 * 24 * 60 * 60 * 1000
    });

    res.json({ accessToken });
  } catch {
    res.status(401).json({ error: 'Invalid refresh token' });
  }
});

// Logout — revoke refresh token
app.post('/auth/logout', async (req, res) => {
  const token = req.cookies.refreshToken;
  if (token) {
    // Add to Redis blacklist until expiry
    await redis.setex(`blacklist:${token}`, 7 * 24 * 3600, '1');
  }
  res.clearCookie('refreshToken');
  res.json({ message: 'Logged out' });
});
```

### ⚠️ Common Mistakes
- Using the same secret for access and refresh tokens
- Storing JWTs in localStorage (XSS vulnerable)
- Not validating `exp` claim — `jwt.verify` does this, but only if you use it
- Forgetting that JWTs can't be invalidated without a blacklist

### 🎯 Interview Tip
> "JWT is stateless — perfect for microservices. For logout or token revocation, I use a Redis blacklist. Access tokens go in memory; refresh tokens in httpOnly cookies to protect against XSS."

---

## Q2 — What are the OWASP Top 10 vulnerabilities relevant to Node.js?

### ✅ Simple Explanation
OWASP Top 10 is the standard list of critical web security risks. As a Node.js developer, these are the ones you must know and protect against.

### 🧠 Deep Dive

| # | Vulnerability | Node.js Example | Fix |
|---|---|---|---|
| A01 | Broken Access Control | No authorization check on routes | Always verify user owns resource |
| A02 | Cryptographic Failures | MD5 password hashing | Use bcrypt/argon2 |
| A03 | Injection | MongoDB query injection | Sanitize inputs, use mongoose |
| A05 | Security Misconfiguration | Default Express headers | Use helmet |
| A07 | Auth Failures | Weak JWTs | Strong secrets, short expiry |
| A09 | Logging Failures | Not logging auth events | Structured logging |

### 💻 Code Example
```js
// A03: NoSQL Injection — MongoDB
// ❌ Vulnerable
app.post('/login', async (req, res) => {
  const user = await User.findOne({
    email: req.body.email,    // attacker sends: { "$gt": "" }
    password: req.body.password
  });
});
// Attack: { "email": { "$gt": "" }, "password": { "$gt": "" } } → logs in as first user!

// ✅ Fix: sanitize with express-mongo-sanitize
const mongoSanitize = require('express-mongo-sanitize');
app.use(mongoSanitize()); // strips $ and . from req.body, params, query

// A02: Cryptographic Failure — password hashing
const bcrypt = require('bcryptjs');

// ❌ Never use: crypto.createHash('md5')
// ✅ Use bcrypt
const SALT_ROUNDS = 12;
async function hashPassword(plaintext) {
  return bcrypt.hash(plaintext, SALT_ROUNDS);
}
async function verifyPassword(plaintext, hash) {
  return bcrypt.compare(plaintext, hash); // constant-time comparison
}

// A05: Security Misconfiguration
const helmet = require('helmet');
app.use(helmet()); // sets 14 security headers:
// X-Content-Type-Options: nosniff
// X-Frame-Options: DENY
// Content-Security-Policy
// Strict-Transport-Security
// X-XSS-Protection

// Remove Express fingerprint
app.disable('x-powered-by'); // or helmet handles it

// A01: Broken Access Control
app.delete('/todos/:id', requireAuth, async (req, res) => {
  const todo = await Todo.findById(req.params.id);
  if (!todo) return res.status(404).json({ error: 'Not found' });
  
  // ✅ Verify ownership — don't just check auth, check authorization
  if (todo.userId.toString() !== req.user.sub) {
    return res.status(403).json({ error: 'Forbidden' });
  }
  
  await todo.deleteOne();
  res.status(204).send();
});
```

### ⚠️ Common Mistakes
- Authentication ≠ Authorization — checking JWT is valid doesn't mean user owns the resource
- Using `==` instead of `===` for secret comparisons — timing attacks

### 🎯 Interview Tip
> "OWASP A01 (Broken Access Control) is #1 for a reason — I always check both authentication (is this a valid user?) AND authorization (does this user own this resource?)."

---

## Q3 — How do you protect against common Node.js attacks?

### ✅ Simple Explanation
Use rate limiting, input sanitization, helmet headers, and HTTPS for most attack vectors. Know what each library does.

### 🧠 Deep Dive
**Attack types and fixes:**

| Attack | Fix |
|---|---|
| Brute force | Rate limiting (`express-rate-limit`) |
| DDoS | Rate limiting + cloud WAF |
| XSS | CSP headers (helmet), sanitize HTML |
| CSRF | SameSite cookies, CSRF tokens |
| SQL/NoSQL injection | Parameterized queries, sanitize |
| ReDoS | Avoid complex regex on user input |
| Path traversal | `path.resolve` + prefix check |

### 💻 Code Example
```js
const rateLimit = require('express-rate-limit');
const slowDown = require('express-slow-down');
const xss = require('xss');

// Rate limiting — per IP
const loginLimiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 min
  max: 5,                    // 5 attempts
  message: 'Too many login attempts',
  standardHeaders: true,
  legacyHeaders: false,
  skipSuccessfulRequests: true,
});
app.post('/auth/login', loginLimiter, loginHandler);

// Slow down before blocking
const speedLimiter = slowDown({
  windowMs: 15 * 60 * 1000,
  delayAfter: 50,
  delayMs: (hits) => hits * 100, // delay grows with each hit
});

// Path traversal prevention
app.get('/files/:filename', (req, res) => {
  const filename = req.params.filename;
  const uploadDir = path.resolve('./uploads');
  const filePath = path.resolve(uploadDir, filename);

  // ✅ Ensure resolved path is within upload directory
  if (!filePath.startsWith(uploadDir)) {
    return res.status(400).json({ error: 'Invalid filename' });
  }

  res.sendFile(filePath);
});

// XSS sanitization for stored content
function sanitizeUserInput(input) {
  return xss(input, {
    whiteList: {}, // no HTML tags allowed
    stripIgnoreTag: true,
  });
}

// CSRF protection for cookie-based auth
const csrf = require('csurf');
const csrfProtection = csrf({ cookie: { httpOnly: true, secure: true } });
app.use(csrfProtection);
app.get('/form', (req, res) => {
  res.render('form', { csrfToken: req.csrfToken() });
});
```

### ⚠️ Common Mistakes
- Rate limiting only on login — also rate limit password reset, OTP endpoints
- Not using `path.resolve` before serving files — path traversal (`../../etc/passwd`)
- Trusting `req.ip` without `trust proxy` configured — returns wrong IP, rate limiting bypassed

### 🎯 Interview Tip
> "I apply rate limiting at the API gateway level (Nginx / AWS API Gateway) AND in the app itself as defense-in-depth. Helmet takes care of headers, but you still need input sanitization and access control."

---

## Q4 — What is OAuth 2.0 and how does it work with Node.js?

### ✅ Simple Explanation
OAuth 2.0 is an authorization framework that lets users grant third-party apps access to their resources without sharing passwords. You see it as "Login with Google/GitHub".

### 🧠 Deep Dive
**OAuth 2.0 Authorization Code Flow** (most secure, for server-side apps):
```
1. User clicks "Login with Google"
2. App redirects to: accounts.google.com/oauth/authorize?client_id=...&redirect_uri=...&scope=email
3. User approves → Google redirects back with: ?code=AUTH_CODE
4. App exchanges code for tokens: POST /token with code + client_secret
5. Google returns: { access_token, refresh_token, id_token }
6. App verifies id_token, creates session
```

**Why not Implicit flow?** Access token in URL = logged in browser history/proxy logs. Deprecated.

**PKCE (Proof Key for Code Exchange):** Adds `code_verifier` + `code_challenge` — prevents auth code interception. Required for SPAs and mobile apps.

### 💻 Code Example
```js
// Using Passport.js with Google OAuth
const passport = require('passport');
const GoogleStrategy = require('passport-google-oauth20').Strategy;

passport.use(new GoogleStrategy({
  clientID: process.env.GOOGLE_CLIENT_ID,
  clientSecret: process.env.GOOGLE_CLIENT_SECRET,
  callbackURL: '/auth/google/callback',
  scope: ['email', 'profile'],
}, async (accessToken, refreshToken, profile, done) => {
  try {
    // Find or create user
    let user = await User.findOne({ googleId: profile.id });
    if (!user) {
      user = await User.create({
        googleId: profile.id,
        email: profile.emails[0].value,
        name: profile.displayName,
      });
    }
    done(null, user);
  } catch (err) {
    done(err);
  }
}));

app.get('/auth/google', passport.authenticate('google'));

app.get('/auth/google/callback',
  passport.authenticate('google', { session: false }),
  (req, res) => {
    const tokens = generateTokens(req.user._id);
    // Set refresh token in cookie, send access token
    res.cookie('refreshToken', tokens.refreshToken, { httpOnly: true, secure: true });
    res.redirect(`${process.env.CLIENT_URL}?token=${tokens.accessToken}`);
  }
);

// Manual OAuth flow (without Passport)
app.get('/auth/github', (req, res) => {
  const state = crypto.randomBytes(16).toString('hex');
  req.session.oauthState = state;
  
  const url = new URL('https://github.com/login/oauth/authorize');
  url.searchParams.set('client_id', process.env.GITHUB_CLIENT_ID);
  url.searchParams.set('redirect_uri', process.env.GITHUB_CALLBACK_URL);
  url.searchParams.set('scope', 'user:email');
  url.searchParams.set('state', state); // CSRF protection
  
  res.redirect(url.toString());
});
```

### ⚠️ Common Mistakes
- Not verifying `state` parameter on callback — enables CSRF on OAuth flow
- Storing `access_token` in your DB without encryption
- Forgetting to validate the `id_token` (JWT) signature from the provider

### 🎯 Interview Tip
> "I always validate the `state` parameter on the OAuth callback to prevent CSRF attacks. For production, I use Passport.js with a session-less JWT approach — no server-side sessions."

---

## ⚡ Quick Revision Summary

- JWT = Header.Payload.Signature; stateless; access (15min) + refresh (7d) pattern
- Refresh tokens in httpOnly cookies; access tokens in memory (not localStorage)
- JWT revocation requires blacklist in Redis (stateless otherwise)
- OWASP: Broken Access Control #1; NoSQL injection, crypto failures, misconfiguration
- `helmet` → security headers; `express-mongo-sanitize` → NoSQL injection; `bcrypt` → passwords
- Rate limit login, password reset, and OTP endpoints
- OAuth 2.0 Authorization Code Flow; validate `state` param; use PKCE for SPAs

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
