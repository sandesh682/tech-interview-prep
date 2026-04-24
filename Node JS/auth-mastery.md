# Authentication & Authorization Mastery

> Single-source revision note for Node.js backend engineers targeting top product companies.
> Goal: Implement → Explain → Design → Defend in interviews.

---

## Table of Contents

- [[#1. Foundations]]
- [[#2. Sessions vs Tokens]]
- [[#3. Cookies — The Transport Layer]]
- [[#4. JWT Deep Dive]]
- [[#5. Access Token vs Refresh Token]]
- [[#6. OAuth 2.0]]
- [[#7. OpenID Connect (OIDC)]]
- [[#8. Security Fundamentals]]
- [[#9. RBAC & Authorization Patterns]]
- [[#10. Node.js Implementation]]
- [[#11. Microservices & API Gateway]]
- [[#12. System Design]]
- [[#13. Okta & Managed Auth]]
- [[#14. LDAP & SAML — Quick View]]
- [[#15. Comparison Tables]]
- [[#16. Visual Flows]]
- [[#17. Learning Roadmap]]
- [[#18. Interview Prep]]

---

## 1. Foundations

### Authentication vs Authorization

| Concept | Question it answers | Example |
|---|---|---|
| **Authentication (AuthN)** | Who are you? | Login with email/password |
| **Authorization (AuthZ)** | What can you do? | Only admins can delete users |

**Why it exists:** Servers are stateless by default. We must prove identity per request and decide what that identity is allowed to do.

**Real-world analogy:**
- AuthN = showing your passport at airport check-in.
- AuthZ = your boarding pass tells you which seat and which lounge you can enter.

**Common mistakes:**
- Mixing both in a single middleware → hard to audit.
- Treating "logged in" as "allowed" — always do an explicit permission check.
- Returning 401 for permission errors (it's 403).

**Interview insight:** Interviewers test if you understand that **AuthN happens once per request (verify identity)**, **AuthZ happens at every resource boundary (verify intent)**.

---

## 2. Sessions vs Tokens

### Session-based (Stateful)

```
Client → POST /login → Server creates session in DB/Redis
Server → Set-Cookie: sid=abc123; HttpOnly; Secure
Client → GET /profile (Cookie: sid=abc123)
Server → Look up sid in store → return data
```

- Server holds state.
- Logout = delete session row. Instant revocation.
- Scales horizontally only with shared store (Redis).

### Token-based (Stateless)

```
Client → POST /login → Server signs a JWT
Client stores token → sends in Authorization: Bearer <token>
Server verifies signature → trusts payload (no DB lookup)
```

- No server state.
- Hard to revoke before expiry without extra infra.
- Easier for microservices and mobile.

**Interview insight:** "Stateless" doesn't mean better. Sessions are simpler, easier to revoke, and Google itself uses session cookies for many products. Tokens win when you need cross-service or cross-domain trust without a shared session store.

---

## 3. Cookies — The Transport Layer

Cookies are **how** the browser carries credentials. They are not auth themselves.

### Critical Flags

| Flag | What it does | Why it matters |
|---|---|---|
| `HttpOnly` | JS cannot read cookie | Stops XSS from stealing token |
| `Secure` | Only sent over HTTPS | Prevents MITM in transit |
| `SameSite=Strict` | Never sent on cross-site requests | Strongest CSRF defense |
| `SameSite=Lax` | Sent on top-level GET navigation | Default in modern browsers |
| `SameSite=None` | Sent on all cross-site requests | Required for third-party use; must pair with `Secure` |
| `Path` / `Domain` | Restricts cookie scope | Limit blast radius |
| `Max-Age` / `Expires` | Lifetime | Short for access, longer for refresh |

### Golden rule

> **Access tokens belong in `HttpOnly` cookies, not `localStorage`.**
> `localStorage` is readable by any script — one XSS = total account takeover.

---

## 4. JWT Deep Dive

### Structure

```
header.payload.signature
```

**Header** (algorithm + type):
```json
{ "alg": "RS256", "typ": "JWT" }
```

**Payload** (claims):
```json
{
  "sub": "user_123",
  "iat": 1730000000,
  "exp": 1730003600,
  "aud": "tiarahub-api",
  "iss": "https://auth.tiarahub.com",
  "roles": ["admin"]
}
```

**Signature**: `HMAC` or `RSA/ECDSA` over `base64(header).base64(payload)`.

### Standard claims (memorize)

| Claim | Meaning |
|---|---|
| `iss` | Issuer |
| `sub` | Subject (user id) |
| `aud` | Audience (which API) |
| `exp` | Expiry (unix seconds) |
| `iat` | Issued at |
| `nbf` | Not before |
| `jti` | Unique token id (for revocation) |

### HS256 vs RS256

| | HS256 | RS256 |
|---|---|---|
| Key | Single shared secret | Private/public key pair |
| Use | Single service | Multi-service / public verification |
| Risk | Anyone with secret can mint tokens | Only auth server can mint |

**Always prefer RS256 in microservices.** Resource services only need the public key.

### Validation checklist (every request)

1. Signature valid?
2. `exp` not passed?
3. `iss` matches expected issuer?
4. `aud` matches your service?
5. `alg` matches expected algorithm? (Reject `none`.)

### Pitfalls

- **`alg: none` attack** — old libraries accept unsigned tokens. Pin the algorithm explicitly.
- **Algorithm confusion** — passing an RSA public key as HMAC secret lets attacker sign with it. Always pin `alg`.
- **Putting sensitive data in payload** — JWT is signed, not encrypted. Anyone can base64-decode it.
- **Long expiry** — without rotation, a leaked token is valid for hours.
- **No audience check** — token meant for service A accepted by service B.

**Interview insight:** If asked "is JWT secure?", answer: "JWT is a format, not a security model. Security depends on signing algorithm, key management, expiry, audience binding, and storage."

---

## 5. Access Token vs Refresh Token

| | Access Token | Refresh Token |
|---|---|---|
| Purpose | Authorize API calls | Get new access token |
| Lifetime | Short (5–15 min) | Long (days/weeks) |
| Sent to | Resource APIs | Only auth server |
| Storage | Memory or HttpOnly cookie | HttpOnly, Secure, SameSite=Strict cookie |
| Revocable | Hard (stateless) | Easy (stored in DB) |

### Refresh Token Rotation

Each refresh issues a **new** refresh token and invalidates the old one.

```
T0: client has RT1
T1: client → /refresh (RT1) → server issues AT2 + RT2, marks RT1 used
T2: if RT1 used again → token reuse detected → revoke entire family → force re-login
```

This detects token theft. If an attacker steals RT1 and uses it, the legitimate user's next refresh will fail and trigger a security event.

### Storage strategy (recommended)

- **Access token** → in-memory variable on frontend, OR `HttpOnly` cookie.
- **Refresh token** → always `HttpOnly`, `Secure`, `SameSite=Strict`, scoped to `/auth/refresh` path.

---

## 6. OAuth 2.0

### What it actually is

OAuth 2.0 is a **delegated authorization framework**. It lets a user grant a third-party app limited access to their resources on another service — **without sharing their password**.

> OAuth is for authorization, NOT authentication. Use **OIDC** if you want to log a user in.

### The Four Roles

| Role | Example |
|---|---|
| **Resource Owner** | The user |
| **Client** | Your app (TiaraHUB) |
| **Authorization Server** | Google's OAuth server |
| **Resource Server** | Google Drive API |

### Grant Types (when to use which)

| Grant Type | Use Case | Status |
|---|---|---|
| **Authorization Code + PKCE** | Web apps, SPAs, mobile | ✅ Modern default |
| **Client Credentials** | Service-to-service (no user) | ✅ For backend integrations |
| **Device Code** | TVs, CLIs, IoT (no browser) | ✅ Used by GitHub CLI, Netflix on TV |
| **Refresh Token** | Renew access without re-login | ✅ Always pair with code grant |
| **Implicit** | SPAs (legacy) | ❌ Deprecated — use PKCE |
| **Password Grant** | Trusted first-party apps | ❌ Deprecated — security risk |

### Authorization Code + PKCE Flow (the one to know)

```
1. User clicks "Login with Google"
2. App generates: code_verifier (random string)
                  code_challenge = SHA256(code_verifier)
3. Redirect to Google:
   /authorize?client_id=...&redirect_uri=...&code_challenge=...&state=xyz
4. User authenticates with Google + grants consent
5. Google redirects back: /callback?code=AUTH_CODE&state=xyz
6. App verifies state (CSRF check)
7. App POSTs to /token with: code + code_verifier
8. Google verifies SHA256(code_verifier) == code_challenge
9. Google returns: { access_token, refresh_token, id_token }
```

**Why PKCE?** Stops authorization code interception attacks. Even if attacker grabs the `code` from the redirect, they can't exchange it without the original `code_verifier`.

**Why `state` parameter?** CSRF protection. Bind it to the user's session.

### Real-world analogy

OAuth = giving a hotel valet a **specific** key card that opens only your car door, valid for 30 minutes, and you can revoke it from your phone. You never hand over your master house key.

### Common mistakes

- Using Implicit flow in 2026 (use PKCE).
- Storing `client_secret` in a browser SPA.
- Not validating `state` → CSRF on login.
- Not validating `redirect_uri` exactly → open redirect attack.

---

## 7. OpenID Connect (OIDC)

OIDC = **Authentication layer on top of OAuth 2.0**.

OAuth 2.0 only gives you an access token (authorization). OIDC adds an **`id_token`** (a JWT) that proves who the user is.

### Key additions over OAuth

| Feature | What it gives you |
|---|---|
| `id_token` | JWT containing user identity claims |
| `userinfo` endpoint | Standardized way to fetch user profile |
| `scope=openid` | Triggers OIDC behavior on top of OAuth |
| Discovery doc | `/.well-known/openid-configuration` |
| Standard claims | `sub`, `email`, `name`, `picture` |

### When to use what

- **Need to call APIs on user's behalf** → OAuth 2.0
- **Need to log user into your app** → OIDC
- **Both** → OIDC (it's a superset)

### Flow (same as OAuth Code+PKCE, just with `scope=openid`)

```
/authorize?response_type=code&scope=openid profile email&...
→ returns code
→ exchange code → { access_token, id_token, refresh_token }
→ verify id_token signature using JWKS
→ user is logged in
```

**Interview insight:** Almost every "Login with X" button in 2026 uses OIDC. Google, Microsoft, Apple, Auth0, Okta — all OIDC providers.

---

## 8. Security Fundamentals

### CSRF (Cross-Site Request Forgery)

**Attack:** Attacker tricks logged-in user's browser into making a state-changing request to your site (browser auto-attaches cookies).

**Defenses:**
1. `SameSite=Lax` or `Strict` cookies (modern default).
2. CSRF tokens (double-submit cookie or synchronizer token).
3. Require custom header (e.g. `X-Requested-With`) — browsers block cross-origin custom headers without CORS preflight.

**JWT in `Authorization` header is naturally CSRF-immune** because the browser doesn't auto-attach it.

### XSS (Cross-Site Scripting)

**Attack:** Attacker injects JS into your page → reads tokens, makes requests as user.

**Defenses:**
1. Output encoding (React/Vue do this by default — don't `dangerouslySetInnerHTML`).
2. `Content-Security-Policy` header.
3. **Tokens in `HttpOnly` cookies** — JS can't touch them.
4. Sanitize user input (DOMPurify for HTML).

### Token Leakage Prevention

- Always HTTPS.
- Short access token TTL (5–15 min).
- Refresh token rotation with reuse detection.
- Bind tokens to client (DPoP, mTLS) for high-value APIs.
- Don't log tokens.
- Don't put tokens in URLs (they end up in server logs and Referer headers).

### Other must-know attacks

| Attack | Defense |
|---|---|
| **Brute force login** | Rate limiting + account lockout + CAPTCHA |
| **Credential stuffing** | Password breach checks (HIBP API), MFA |
| **Session fixation** | Regenerate session ID on login |
| **Open redirect** | Whitelist `redirect_uri` exactly |
| **Timing attacks on password compare** | Use `crypto.timingSafeEqual` / bcrypt's compare |
| **JWT `alg: none`** | Pin algorithm in verify call |

---

## 9. RBAC & Authorization Patterns

### RBAC (Role-Based Access Control)

```
User → has Roles → have Permissions → on Resources
```

**Schema:**
```
users(id, email)
roles(id, name)
permissions(id, action, resource)
user_roles(user_id, role_id)
role_permissions(role_id, permission_id)
```

### ABAC (Attribute-Based)

Decisions based on attributes: `user.department === resource.department && time < 18:00`.
More flexible, harder to audit.

### ReBAC (Relationship-Based)

Used by Google Zanzibar. "Can user X view doc Y?" → graph traversal.
Used by Notion, Figma, GitHub for fine-grained sharing.

### Quick decision guide

| Scale | Pattern |
|---|---|
| Small app, few roles | RBAC |
| Multi-tenant SaaS, complex rules | ABAC |
| Document/resource sharing | ReBAC |

---

## 10. Node.js Implementation

Production-style examples. Express + modern patterns.

### 10.1 Project Structure

```
src/
  config/
    env.js
    jwt.js
  modules/
    auth/
      auth.controller.js
      auth.service.js
      auth.routes.js
      auth.middleware.js
      strategies/
        google.strategy.js
    users/
      user.model.js
      user.service.js
  middleware/
    requireAuth.js
    requireRole.js
    errorHandler.js
  utils/
    tokens.js
    cookies.js
    crypto.js
  app.js
  server.js
```

### 10.2 Token Utilities

```js
import jwt from 'jsonwebtoken';
import { randomBytes, createHash } from 'crypto';
import { config } from '../config/jwt.js';

export const signAccessToken = (payload) =>
  jwt.sign(payload, config.accessPrivateKey, {
    algorithm: 'RS256',
    expiresIn: '15m',
    issuer: config.issuer,
    audience: config.audience,
  });

export const verifyAccessToken = (token) =>
  jwt.verify(token, config.accessPublicKey, {
    algorithms: ['RS256'],
    issuer: config.issuer,
    audience: config.audience,
  });

export const generateRefreshToken = () => {
  const raw = randomBytes(64).toString('hex');
  const hash = createHash('sha256').update(raw).digest('hex');
  return { raw, hash };
};
```

### 10.3 Cookie Helper

```js
const baseCookieOptions = {
  httpOnly: true,
  secure: process.env.NODE_ENV === 'production',
  sameSite: 'strict',
};

export const setAccessCookie = (res, token) =>
  res.cookie('access_token', token, {
    ...baseCookieOptions,
    maxAge: 15 * 60 * 1000,
    path: '/',
  });

export const setRefreshCookie = (res, token) =>
  res.cookie('refresh_token', token, {
    ...baseCookieOptions,
    maxAge: 7 * 24 * 60 * 60 * 1000,
    path: '/auth/refresh',
  });

export const clearAuthCookies = (res) => {
  res.clearCookie('access_token', { path: '/' });
  res.clearCookie('refresh_token', { path: '/auth/refresh' });
};
```

### 10.4 Auth Service (login + refresh + rotation)

```js
import bcrypt from 'bcrypt';
import { signAccessToken, generateRefreshToken } from '../../utils/tokens.js';
import { UserRepo } from '../users/user.repo.js';
import { RefreshTokenRepo } from './refreshToken.repo.js';

export const AuthService = {
  async login(email, password) {
    const user = await UserRepo.findByEmail(email);
    if (!user) throw new AppError('INVALID_CREDENTIALS', 401);

    const ok = await bcrypt.compare(password, user.passwordHash);
    if (!ok) throw new AppError('INVALID_CREDENTIALS', 401);

    return AuthService.issueTokenPair(user);
  },

  async issueTokenPair(user, familyId = null) {
    const accessToken = signAccessToken({
      sub: user.id,
      roles: user.roles,
    });

    const { raw, hash } = generateRefreshToken();
    const family = familyId ?? crypto.randomUUID();

    await RefreshTokenRepo.create({
      userId: user.id,
      tokenHash: hash,
      familyId: family,
      expiresAt: new Date(Date.now() + 7 * 24 * 60 * 60 * 1000),
    });

    return { accessToken, refreshToken: raw, familyId: family };
  },

  async rotateRefreshToken(rawToken) {
    const hash = sha256(rawToken);
    const stored = await RefreshTokenRepo.findByHash(hash);

    if (!stored) throw new AppError('INVALID_REFRESH', 401);

    if (stored.usedAt) {
      await RefreshTokenRepo.revokeFamily(stored.familyId);
      throw new AppError('TOKEN_REUSE_DETECTED', 401);
    }

    if (stored.expiresAt < new Date()) {
      throw new AppError('REFRESH_EXPIRED', 401);
    }

    await RefreshTokenRepo.markUsed(stored.id);
    const user = await UserRepo.findById(stored.userId);
    return AuthService.issueTokenPair(user, stored.familyId);
  },

  async logout(rawToken) {
    if (!rawToken) return;
    const hash = sha256(rawToken);
    const stored = await RefreshTokenRepo.findByHash(hash);
    if (stored) await RefreshTokenRepo.revokeFamily(stored.familyId);
  },
};
```

### 10.5 Middleware: requireAuth

```js
import { verifyAccessToken } from '../utils/tokens.js';

export const requireAuth = (req, res, next) => {
  try {
    const token =
      req.cookies?.access_token ??
      req.headers.authorization?.replace(/^Bearer\s+/i, '');

    if (!token) return res.status(401).json({ error: 'UNAUTHENTICATED' });

    const payload = verifyAccessToken(token);
    req.user = { id: payload.sub, roles: payload.roles ?? [] };
    next();
  } catch {
    res.status(401).json({ error: 'INVALID_TOKEN' });
  }
};
```

### 10.6 Middleware: requireRole

```js
export const requireRole = (...allowed) => (req, res, next) => {
  const roles = req.user?.roles ?? [];
  const ok = roles.some((r) => allowed.includes(r));
  if (!ok) return res.status(403).json({ error: 'FORBIDDEN' });
  next();
};
```

Usage:
```js
router.delete('/users/:id', requireAuth, requireRole('admin'), UserController.remove);
```

### 10.7 Auth Controller

```js
import { AuthService } from './auth.service.js';
import { setAccessCookie, setRefreshCookie, clearAuthCookies } from '../../utils/cookies.js';

export const AuthController = {
  async login(req, res) {
    const { email, password } = req.body;
    const { accessToken, refreshToken } = await AuthService.login(email, password);
    setAccessCookie(res, accessToken);
    setRefreshCookie(res, refreshToken);
    res.json({ ok: true });
  },

  async refresh(req, res) {
    const raw = req.cookies?.refresh_token;
    if (!raw) return res.status(401).json({ error: 'NO_REFRESH' });
    const { accessToken, refreshToken } = await AuthService.rotateRefreshToken(raw);
    setAccessCookie(res, accessToken);
    setRefreshCookie(res, refreshToken);
    res.json({ ok: true });
  },

  async logout(req, res) {
    await AuthService.logout(req.cookies?.refresh_token);
    clearAuthCookies(res);
    res.json({ ok: true });
  },
};
```

### 10.8 OAuth/OIDC Login with Google (no Passport — using openid-client)

`openid-client` is the modern, spec-compliant choice. Passport.js has older patterns and many unmaintained strategies.

```js
import { Issuer, generators } from 'openid-client';

let client;
export const initGoogleClient = async () => {
  const google = await Issuer.discover('https://accounts.google.com');
  client = new google.Client({
    client_id: process.env.GOOGLE_CLIENT_ID,
    client_secret: process.env.GOOGLE_CLIENT_SECRET,
    redirect_uris: [process.env.GOOGLE_REDIRECT_URI],
    response_types: ['code'],
  });
};

export const GoogleAuthController = {
  start(req, res) {
    const code_verifier = generators.codeVerifier();
    const code_challenge = generators.codeChallenge(code_verifier);
    const state = generators.state();

    req.session.oauth = { code_verifier, state };

    const url = client.authorizationUrl({
      scope: 'openid email profile',
      code_challenge,
      code_challenge_method: 'S256',
      state,
    });
    res.redirect(url);
  },

  async callback(req, res) {
    const params = client.callbackParams(req);
    const { code_verifier, state } = req.session.oauth ?? {};

    const tokenSet = await client.callback(
      process.env.GOOGLE_REDIRECT_URI,
      params,
      { code_verifier, state }
    );

    const claims = tokenSet.claims();
    const user = await UserService.upsertFromOidc({
      provider: 'google',
      providerId: claims.sub,
      email: claims.email,
      name: claims.name,
      picture: claims.picture,
    });

    const { accessToken, refreshToken } = await AuthService.issueTokenPair(user);
    setAccessCookie(res, accessToken);
    setRefreshCookie(res, refreshToken);
    res.redirect('/dashboard');
  },
};
```

### 10.9 Routes

```js
import { Router } from 'express';

const router = Router();

router.post('/auth/login', AuthController.login);
router.post('/auth/refresh', AuthController.refresh);
router.post('/auth/logout', AuthController.logout);

router.get('/auth/google', GoogleAuthController.start);
router.get('/auth/google/callback', GoogleAuthController.callback);

export default router;
```

### 10.10 Password Hashing

```js
import bcrypt from 'bcrypt';

export const hashPassword = (plain) => bcrypt.hash(plain, 12);
export const verifyPassword = (plain, hash) => bcrypt.compare(plain, hash);
```

Use **bcrypt cost 12+** in 2026, or **argon2id** for new systems (memory-hard, GPU-resistant).

---

## 11. Microservices & API Gateway

### Pattern 1: Centralized Auth Service

```
            ┌─────────────┐
Client ───▶│ API Gateway │──▶ verifies JWT (public key)
            └─────┬───────┘
                  │ forwards request + user context
                  ▼
        ┌─────────┴──────────┐
        ▼                    ▼
  Orders Service        Inventory Service
```

- Auth service issues JWTs (RS256).
- Gateway validates them using the public key (JWKS endpoint).
- Downstream services trust the gateway's injected headers (`X-User-Id`, `X-Roles`) — but **must not** be reachable from outside the mesh.

### Pattern 2: Token Propagation

Each service re-validates the JWT. Safer (zero trust). Slightly slower but acceptable with cached JWKS.

### Pattern 3: BFF (Backend-For-Frontend)

Frontend talks to a BFF. BFF holds the session/refresh token and uses access tokens internally. Tokens never reach the browser.

> **Use BFF for SPAs.** It's the OWASP-recommended pattern in 2026.

### Service-to-service auth

- **Client Credentials grant** → service A gets a token for service B.
- **mTLS** → mutual TLS for stronger identity (used inside service mesh like Istio/Linkerd).
- **SPIFFE/SPIRE** → workload identity at scale.

---

## 12. System Design

### Designing a Centralized Auth Service

**Components:**
1. **Auth API** — `/login`, `/refresh`, `/logout`, `/oauth/*`
2. **User store** (Postgres) — users, credentials, MFA
3. **Refresh token store** (Postgres) — for revocation, rotation
4. **JWKS endpoint** — `/.well-known/jwks.json` for public keys
5. **Cache** (Redis) — rate limiting, session lookups, revocation lists
6. **Event bus** — publish `user.logged_out`, `user.deleted` for downstream invalidation

### Stateless vs Stateful tradeoffs

| | Stateless (JWT) | Stateful (Session) |
|---|---|---|
| Scalability | ✅ Trivial horizontal scale | Needs shared store |
| Revocation | ❌ Hard (use short TTL + denylist) | ✅ Delete row |
| Network overhead | Larger headers | Tiny session ID |
| Cross-service | ✅ No shared DB needed | Needs shared cache |
| Mobile/SPA | ✅ Natural fit | Works with cookies |

**Hybrid approach (Google's model):**
- Long-lived session in DB (cookie-based).
- Short-lived JWT issued from session for API calls.
- Best of both: instant revocation + stateless API calls.

### Token storage strategies

| Where | Pros | Cons |
|---|---|---|
| `localStorage` | Easy | XSS = total compromise. **Avoid.** |
| `sessionStorage` | Cleared on tab close | Same XSS risk |
| Memory (JS variable) | Safe from XSS | Lost on refresh |
| `HttpOnly` cookie | Safe from XSS | CSRF risk → mitigate with SameSite |
| **BFF + HttpOnly cookie** | Best | More infra |

### Handling logout, revocation, expiry

**Logout:**
1. Clear cookies on client.
2. Revoke refresh token family in DB.
3. (Optional) Add access token `jti` to short-lived denylist in Redis (TTL = remaining token life).

**Forced logout (e.g., password change):**
1. Increment `tokenVersion` on user record.
2. Bake `tokenVersion` into JWT payload.
3. Middleware rejects if `payload.tokenVersion !== user.tokenVersion`.
4. All existing tokens invalid instantly.

### Scaling auth

- **Cache JWKS** in services (TTL ~1 hour) — verification is local, no network call.
- **Key rotation** — JWKS supports multiple `kid`s; rotate keys without breaking active tokens.
- **Rate limit** by IP + by account (Redis sliding window).
- **Geo-distribute** auth servers; refresh tokens stored in regional DBs.
- **Async invalidation** — publish events on logout/password change; services consume and update local denylists.

---

## 13. Okta & Managed Auth

### What Okta does

Hosts your entire identity layer:
- User store + login UI
- OIDC/OAuth/SAML provider
- MFA, SSO, social login
- User lifecycle (provisioning, deprovisioning)
- Audit logs, compliance reports

### Build vs Buy

| Build your own | Use Okta/Auth0/Cognito |
|---|---|
| Full control | Speed to market |
| No vendor lock-in | Compliance baked in (SOC2, GDPR) |
| No per-user cost | Per-user pricing scales fast |
| You own security incidents | They own incidents |
| Months of work | Days of work |

**Rule of thumb:** Use a managed provider unless auth IS your product (e.g., you're building Auth0). For a B2B SaaS, Okta/Auth0/WorkOS is almost always the right call.

### Basic Okta integration (OIDC, Node.js)

```js
import { Issuer } from 'openid-client';

const okta = await Issuer.discover(`https://${process.env.OKTA_DOMAIN}/oauth2/default`);

const client = new okta.Client({
  client_id: process.env.OKTA_CLIENT_ID,
  client_secret: process.env.OKTA_CLIENT_SECRET,
  redirect_uris: [process.env.OKTA_REDIRECT_URI],
  response_types: ['code'],
});
```

The flow is identical to the Google example above — that's the point of OIDC standardization.

---

## 14. LDAP & SAML — Quick View

### LDAP (Lightweight Directory Access Protocol)

- **What:** Protocol for querying directory services (think corporate user database).
- **Where:** Microsoft Active Directory, OpenLDAP. Enterprise intranets.
- **When you'll see it:** Internal tools at large companies. "Login with your corporate ID."
- **Modern alternative:** SCIM for provisioning, OIDC for login. LDAP is mostly legacy.

### SAML (Security Assertion Markup Language)

- **What:** XML-based SSO protocol. Older sibling of OIDC.
- **Where:** Enterprise SSO. Salesforce, Workday, banking apps.
- **Flow:** User → Service Provider → redirected to Identity Provider → signed XML assertion → back to SP.
- **Why it persists:** Enterprises invested heavily in SAML IdPs (Okta, ADFS, PingFederate).
- **Modern alternative:** OIDC. Cleaner, JSON-based, mobile-friendly.

**Interview answer:** "SAML and OIDC solve the same problem (federated SSO). SAML is XML and SOAP-era; OIDC is JSON and REST-era. New systems use OIDC; enterprise integrations may still need SAML support."

---

## 15. Comparison Tables

### OAuth 2.0 vs OIDC

| | OAuth 2.0 | OIDC |
|---|---|---|
| Purpose | Authorization (delegated access) | Authentication (login) |
| Token type | Access token (opaque or JWT) | + ID token (always JWT) |
| User info | Not standardized | `/userinfo` endpoint, standard claims |
| Scope trigger | Any scope | Must include `openid` |
| Built on | — | OAuth 2.0 |

### Session vs JWT

| | Session | JWT |
|---|---|---|
| State location | Server | Client |
| Revocation | Instant (delete row) | Hard (denylist or short TTL) |
| Scalability | Needs shared store | Stateless |
| Payload size | Tiny (just ID) | ~500B–2KB |
| Cross-domain | Hard | Easy |
| Best for | Monolith web apps | APIs, microservices, mobile |

### SAML vs OIDC

| | SAML | OIDC |
|---|---|---|
| Era | 2005 | 2014 |
| Format | XML | JSON |
| Transport | SOAP/HTTP POST | REST/HTTP redirects |
| Mobile-friendly | ❌ Painful | ✅ Native |
| Token type | XML assertion | JWT |
| Use today | Enterprise SSO (legacy) | Everything new |

### Stateful vs Stateless Auth

| | Stateful | Stateless |
|---|---|---|
| Server stores session | ✅ | ❌ |
| Token = identity proof | Session ID (opaque) | Self-contained JWT |
| Revoke before expiry | Easy | Needs extra infra |
| Horizontal scaling | Needs shared store | Free |
| Network calls per request | 1 (DB/Redis lookup) | 0 (signature verify) |

---

## 16. Visual Flows

### OAuth 2.0 Authorization Code + PKCE

```
┌────────┐         ┌────────────┐         ┌────────────┐
│ User   │         │   Client   │         │ Auth Server│
│        │         │  (Your App)│         │  (Google)  │
└───┬────┘         └─────┬──────┘         └─────┬──────┘
    │                    │                      │
    │ Click "Login"      │                      │
    ├───────────────────▶│                      │
    │                    │ Generate verifier    │
    │                    │ + challenge          │
    │                    │                      │
    │                    │ Redirect with        │
    │                    │ challenge + state    │
    │◀───────────────────┤                      │
    │                                           │
    │ Login + consent                           │
    ├──────────────────────────────────────────▶│
    │                                           │
    │ Redirect with code + state                │
    │◀──────────────────────────────────────────┤
    │                                           │
    │ Forward code       │                      │
    ├───────────────────▶│                      │
    │                    │                      │
    │                    │ POST code + verifier │
    │                    ├─────────────────────▶│
    │                    │                      │
    │                    │ Verify SHA256        │
    │                    │ Return tokens        │
    │                    │◀─────────────────────┤
    │                    │                      │
    │ Set cookies, login │                      │
    │◀───────────────────┤                      │
```

### OIDC (same + ID token)

```
... same as OAuth ...
Token response: {
  access_token,    ← call APIs
  id_token (JWT),  ← prove identity
  refresh_token    ← renew
}
↓
Client verifies id_token signature using JWKS
Extracts sub, email, name → user is logged in
```

### JWT Request Flow

```
┌────────┐                     ┌────────┐                 ┌──────────┐
│ Client │                     │   API  │                 │  No DB!  │
└───┬────┘                     └────┬───┘                 └──────────┘
    │ GET /orders                   │
    │ Authorization: Bearer <jwt>   │
    ├──────────────────────────────▶│
    │                               │ Verify signature (cached pub key)
    │                               │ Check exp, iss, aud
    │                               │ Extract sub, roles
    │                               │
    │ 200 [orders]                  │
    │◀──────────────────────────────┤
```

### Refresh Token Rotation

```
T0: Login
    Server → AT1 (15min) + RT1 (7d), family=F1
    DB: { hash(RT1), family=F1, used=false }

T1: AT1 expires → POST /refresh with RT1
    Server → looks up hash(RT1) → not used → mark used
    Server → AT2 + RT2, family=F1
    DB: { hash(RT1), used=true } + { hash(RT2), used=false }

T2 (attacker): replays RT1
    Server → looks up hash(RT1) → used=true!
    Server → REVOKE entire family F1
    Server → 401, force re-login
    Real user's next refresh fails → user notices → password change
```

---

## 17. Learning Roadmap

### Week 1 — Foundations

| Day | Learn | Build |
|---|---|---|
| 1 | AuthN vs AuthZ, sessions vs tokens | — |
| 2 | Cookies deep dive, security flags | Express app with session cookies (Redis store) |
| 3 | JWT structure, signing, validation | JWT mint + verify utility, test with jwt.io |
| 4 | Access + Refresh, rotation | Build full login + refresh + logout |
| 5 | CSRF, XSS, secure cookies | Add CSRF protection to your app |
| 6 | RBAC | Add roles + `requireRole` middleware |
| 7 | Revise + write a blog draft | — |

### Week 2 — OAuth & OIDC

| Day | Learn | Build |
|---|---|---|
| 8 | OAuth roles, grants, why PKCE | Read OAuth 2.0 RFC 6749 sections 1–4 |
| 9 | Auth Code + PKCE flow | Implement Google login with `openid-client` |
| 10 | OIDC, ID tokens, JWKS | Verify Google's id_token manually |
| 11 | Refresh tokens with OAuth | Add refresh flow for Google tokens |
| 12 | Common attacks: open redirect, state | Audit your implementation |
| 13 | Compare OAuth, OIDC, SAML | Make your own comparison cheatsheet |
| 14 | Revise | — |

### Week 3 — System Design

| Day | Learn | Build |
|---|---|---|
| 15 | Microservices auth patterns | Sketch architecture for TiaraHUB |
| 16 | Centralized auth service design | Design doc: auth service for 10M users |
| 17 | Token storage, BFF pattern | Refactor frontend to use BFF |
| 18 | Revocation strategies | Implement `tokenVersion` mechanism |
| 19 | Scaling, JWKS rotation | Add key rotation to your auth service |
| 20 | Service-to-service (mTLS, client creds) | Implement client credentials flow |
| 21 | Revise | — |

### Week 4 — Polish + Interview

| Day | Activity |
|---|---|
| 22 | Mock interview: "Design Login with Google" |
| 23 | Mock: "How would you scale auth to 100M users?" |
| 24 | Mock: "Walk me through JWT validation" |
| 25 | Read Google's identity blog posts |
| 26 | Read Auth0 blog: top 10 articles |
| 27 | Build: full auth service with Okta integration as comparison |
| 28 | Final revision of this note |

---

## 18. Interview Prep

### Common Questions (with crisp answers)

**Q: What's the difference between authentication and authorization?**
> AuthN proves who you are (login). AuthZ decides what you can do (permissions). One happens at the perimeter, the other at every resource boundary.

**Q: How does JWT work?**
> Three base64 parts: header (algorithm), payload (claims like sub, exp, aud), signature. Server signs with a key; resource servers verify the signature and trust the claims. Stateless: no DB lookup needed per request.

**Q: Where should I store JWTs in a browser?**
> Never `localStorage` — XSS reads it. Best is HttpOnly cookies with SameSite=Strict. Even better: use a BFF so tokens never reach the browser at all.

**Q: How do you revoke a JWT?**
> JWTs are stateless, so true revocation requires extra infra. Options: (1) short TTL + refresh token rotation, (2) denylist by `jti` in Redis with TTL = remaining token life, (3) `tokenVersion` in payload that you bump on logout/password change.

**Q: Walk me through OAuth Authorization Code + PKCE.**
> Client generates a random `code_verifier` and sends `SHA256(code_verifier)` as the challenge. User logs in at the auth server, which returns a `code` to the client's redirect URI. Client exchanges `code + verifier` for tokens. Auth server verifies the SHA matches. Stops code interception attacks because the attacker doesn't have the original verifier.

**Q: OAuth vs OIDC?**
> OAuth is authorization — get a token to call APIs on a user's behalf. OIDC adds an `id_token` (JWT) on top, which proves identity. Use OAuth for "access my Drive files," use OIDC for "log me in."

**Q: How do you handle auth in microservices?**
> Centralized auth service issues RS256 JWTs. API Gateway validates them at the edge using cached JWKS. Downstream services either re-validate (zero trust) or trust gateway-injected headers if they're not externally reachable. For service-to-service, use client credentials grant or mTLS.

**Q: How do you prevent CSRF?**
> Three layers: (1) `SameSite=Lax` or `Strict` cookies, (2) CSRF token (double-submit cookie or synchronizer), (3) prefer JWT in `Authorization` header for APIs since browsers don't auto-attach those.

**Q: Why is `localStorage` bad for tokens?**
> It's accessible to any JavaScript on your page. One XSS — a vulnerable npm package, a third-party script — and the attacker exfiltrates the token. HttpOnly cookies are immune to JS access.

**Q: How would you design auth for 100M users?**
> Centralized auth service with stateless JWT issuance. Auth DB is sharded by user_id. Refresh tokens in Postgres for revocability. JWKS cached at every service (TTL ~1h). Redis for rate limiting and short-TTL denylist. Async events on logout/password change consumed by services to update local denylists. Geo-distributed auth servers with regional refresh token storage.

### Scenario-Based Questions

**"User reports they're still logged in after changing password from another device."**
> They're using the old JWT until it expires. Fix: bump `tokenVersion` on the user record on password change. Bake `tokenVersion` into JWTs. Middleware rejects mismatches. All old tokens dead instantly.

**"Mobile app shows users logged out randomly."**
> Likely refresh token rotation race condition: app makes two parallel requests, both trigger refresh, second one trips reuse detection. Fix: serialize refresh calls on the client (single-flight pattern), or use grace period in rotation.

**"Login works in Chrome but not Safari (cross-domain)."**
> Safari ITP (Intelligent Tracking Prevention) blocks third-party cookies. If your auth server is on a different domain than your app, cookies are treated as third-party. Fix: use same parent domain (auth.yourdomain.com + app.yourdomain.com), or move auth into a BFF on the same origin.

**"Attacker has stolen a user's refresh token."**
> If you implemented rotation with reuse detection: the moment either the attacker or the legitimate user uses it next, the other's next call will fail. Detect, revoke entire family, force re-login, alert user. Without rotation, you're relying on TTL — much weaker.

**"How would you migrate from session auth to JWT without downtime?"**
> Dual-read phase: middleware accepts both. New logins issue JWTs. Old sessions kept until natural expiry. Once session traffic drops to zero, remove session code. Total: 2× max session lifetime.

### Tricky Edge Cases

- **Clock skew:** servers in different timezones/NTP drift can reject just-issued tokens. Allow `clockTolerance: 30s` in JWT verify.
- **JWKS rotation:** never trust a single key. Cache the full keyset, look up by `kid` header.
- **Logout from all devices:** session model = delete all rows. JWT model = bump `tokenVersion`.
- **Email change:** treat like password change — invalidate all sessions/tokens.
- **MFA bypass via refresh:** ensure MFA is checked on refresh too if step-up auth is required.
- **Race condition on signup:** two parallel requests with same email → unique constraint on DB, handle the duplicate error gracefully.
- **Cookie size limits:** 4KB per cookie. Don't stuff JWTs with large claims; use opaque tokens and look up details.
- **iframe embedding:** `SameSite=Strict` breaks iframes. Need `None` + `Secure` (and accept the CSRF tradeoff or use anti-CSRF tokens).

### Red Flags Interviewers Hate

- "I store JWT in localStorage because it's easier."
- "I use HS256 in microservices and share the secret."
- "Refresh tokens never expire."
- "I use Implicit flow because PKCE is complicated."
- Confusing AuthN with AuthZ.
- Saying JWTs are encrypted (they're signed; payload is readable).

### Green Flags Interviewers Love

- Mentioning **PKCE for all public clients** (SPA + mobile).
- Knowing **refresh token rotation with reuse detection**.
- Distinguishing **OAuth (authorization) vs OIDC (authentication)**.
- Citing **JWKS** for key rotation.
- Suggesting **BFF pattern** for SPAs.
- Discussing **revocation tradeoffs** honestly instead of pretending JWTs solve everything.

---

## Quick Reference Card (for last-minute revision)

```
AUTH STACK PICK LIST
─────────────────────
Web (server-rendered)  → Session cookies + Redis
SPA                    → BFF + HttpOnly cookies
Mobile                 → OAuth Code+PKCE + RT rotation
Microservices          → JWT RS256 + JWKS + central auth svc
Service-to-service     → Client Credentials grant or mTLS
Enterprise SSO         → OIDC (or SAML if forced)
Build vs Buy           → Buy (Okta/Auth0/WorkOS) unless auth IS your product

ALWAYS                 NEVER
─────────              ─────
HTTPS                  Implicit grant
HttpOnly + Secure      localStorage for tokens
SameSite=Strict        HS256 in microservices
RS256 + JWKS           Long-lived access tokens
Short access TTL       JWT alg=none acceptance
RT rotation            Skip state param check
Pin alg in verify      Skip aud claim check
Verify iss/aud/exp     Trust unsigned tokens
bcrypt cost ≥12        Plaintext passwords (ever)
Rate limit login       Detailed login errors
                       (use generic "invalid credentials")
```
