# Area: Security Analysis (Node)

Reference for Phase 3.3. Produces a structured security report covering authentication, authorization, API security, secrets management, and security testing.

**Core principle**: Understand the security model before you write code. Node apps are particularly prone to misconfigurations because security is *composed* (middleware order, manual checks) rather than declared.

## Priority Tiers (within this area)

| Step | Priority | Rationale |
|---|---|---|
| Step 2: Authentication | **P0** | Must know how auth works to make any API call |
| Step 3.1: Authorization (route-level) | **P0** | Must know which endpoints are public vs. protected |
| Step 4.6: Secrets Management | **P0** | Must know how to configure secrets locally |
| Step 6: Local Dev Auth | **P0** | Must authenticate locally to develop |
| Step 3.2–3.3: Method-Level / Custom Authorization | **P1** | For writing secured code |
| Step 4.1–4.5: CORS, Headers, Rate limiting, Sessions, CSRF | **P1** | Important but not day-1 blockers |
| Step 5: Security Testing | **P1** | Need to write security-aware tests |
| Step 4.7: Custom Middleware Deep Dive | **P2** | Only for security-focused work |

**Quick mode**: Steps 1, 2.1 (auth mechanism), 4.6 (secrets), 6 (local dev auth). Skip everything else.

## Ledger Read-Before

Check the ledger. If 3.2 already identified the auth library and the middleware stack, start from that finding. Pull auth-related dependencies from the build file analysis.

Note `Framework`, `Edge runtime target`, `Decorator usage` from Project Identity — they affect how security is composed.

## Symbol Tracing

For Node, auth is usually middleware. Useful greps:
- Auth middleware uses: `passport.authenticate`, `requireAuth`, `authMiddleware`, `verifyJwt`, `withAuth`.
- Route-level guards: `@UseGuards` (NestJS), per-route arrays in Express/Fastify route options.
- User attached to request: `req.user`, `ctx.state.user`, `c.var.user` (Hono) — where this is populated is the auth boundary.

## Step 1: Identify the Security Stack — P0

If the stack was already identified in 3.2, skip to Step 2.

### 1.1 Auth libraries (by dependency)

| Dependency | Auth Mechanism |
|---|---|
| `passport`, `passport-jwt`, `passport-local`, `passport-google-oauth20`, ... | Passport (most common, strategy-based) |
| `jsonwebtoken` | Manual JWT (sign/verify) |
| `jose` | Modern JWT/JWE/JWS (also works in edge runtimes) |
| `@fastify/jwt`, `@fastify/auth` | Fastify-native JWT |
| `@nestjs/passport`, `@nestjs/jwt` | NestJS auth modules (wraps Passport) |
| `lucia` | Modern session-based auth library |
| `next-auth` / `@auth/core` | Next.js (and other) authentication |
| `clerk-sdk-node`, `@clerk/clerk-sdk-node` | Clerk (hosted auth) |
| `@auth0/auth0-spa-js`, `auth0-js`, `express-oauth2-jwt-bearer` | Auth0 (hosted) |
| `firebase-admin` | Firebase Auth |
| `@supabase/supabase-js` | Supabase Auth |
| `oidc-client`, `openid-client` | Generic OIDC |
| `express-session`, `cookie-session`, `@fastify/session` | Server-side sessions |
| `iron-session`, `iron-webcrypto` | Encrypted cookie sessions (edge-friendly) |
| `csurf` (legacy), `@fastify/csrf-protection` | CSRF |
| `helmet` | Security headers |
| `express-rate-limit`, `@fastify/rate-limit`, `rate-limiter-flexible` | Rate limiting |

### 1.2 Auth-related env vars

Grep `process.env` for: `JWT_SECRET`, `JWT_PUBLIC_KEY`, `SESSION_SECRET`, `COOKIE_SECRET`, `OAUTH_*`, `AUTH0_*`, `CLERK_*`, `SUPABASE_*`, `OIDC_*`.

### 1.3 Find the auth configuration

- Express/Fastify/Koa: Look in `src/middleware/`, `src/auth/`, `src/lib/auth.ts`.
- NestJS: `auth.module.ts`, guards, strategies — typically in `src/auth/`.
- Next.js: `middleware.ts` at root + `[...auth]` route handlers + `next-auth.config.ts`.

**Present findings**: Auth library, mechanism (session vs. JWT vs. OAuth/OIDC vs. external IdP), identity provider, where config lives.

## Step 2: Analyze Authentication — P0

### 2.1 Authentication Mechanism — P0

Identify the active flow:

| Mechanism | Detection | Key Analysis |
|---|---|---|
| **JWT (Bearer token)** | `jsonwebtoken` / `jose`, header parsing | Signing alg, secret/key source, claims, expiration, refresh token flow |
| **Session-based** | `express-session` / `iron-session` / `lucia` | Session store (Redis / DB / cookie), expiration, secure cookie flags |
| **OAuth / OIDC** | `openid-client`, `passport-google-oauth20`, etc. | Provider, client ID/secret env var, redirect URI, scopes |
| **External IdP (managed)** | Clerk / Auth0 / Supabase / Firebase / NextAuth | Provider config, webhook endpoints, JWKS endpoint |
| **API key** | Custom middleware checking headers | Key source (DB? env? config?), validation, rotation |
| **mTLS** | TLS client certs configured at server or proxy | Cert source, CN/SAN validation |
| **Basic auth** | `Authorization: Basic ...` header parsing | User store, hashing |

### 2.2 JWT analysis (if applicable) — P1

| What to Find | Where to Look |
|---|---|
| Signing algorithm | `jsonwebtoken` `verify(..., {algorithms: [...]})`; flag if `none` is allowed |
| Key management | Symmetric secret in env vs. JWKS URI vs. local public key |
| Claims validation | `iss` (issuer), `aud` (audience), `exp` (expiration), `nbf` (not-before) — must all be checked |
| Token extraction | Authorization header vs. cookie vs. query param |
| Token refresh | Separate refresh token? Sliding expiration? |

**Anti-patterns to flag immediately**:
- `jwt.verify(token, secret)` without `algorithms` array → `none` attack possible.
- Hardcoded `JWT_SECRET` string in source.
- Tokens with no expiration.
- JWT secrets that are short (< 256 bits for HS256).

### 2.3 Session analysis (if applicable) — P1

- Session store: in-memory (default Express, **bad for prod**), Redis (`connect-redis`), DB-backed, encrypted-cookie (`iron-session`).
- Cookie flags: `httpOnly`, `secure`, `sameSite`, `domain`, `maxAge`.
- Session fixation protection: regenerated on login?
- Concurrent sessions: allowed or revoked?

### 2.4 Password handling (if local user management)

- Hashing library: `bcrypt`, `bcryptjs`, `argon2` (preferred), `scrypt` (Node native). **Flag SHA-1, SHA-256, MD5 — these are not for passwords.**
- Salt rounds: bcrypt cost factor (10 minimum, 12 preferred); argon2 memory cost.
- Password policy enforcement: validation on input.
- Reset flow: token expiration, one-time use.

## Step 3: Analyze Authorization — P0/P1

### 3.1 Route-level (URL-based) — P0

Map every route to its access rule. The "where" varies:
- **Express**: middleware on the route or router: `router.get('/admin', requireAdmin, handler)`.
- **Fastify**: `preHandler` on the route options, or per-route auth options.
- **NestJS**: `@UseGuards(JwtAuthGuard, RolesGuard)` on controller or method.
- **Next.js**: `middleware.ts` at project root using `matcher` config; or per-route checks inside the handler.
- **tRPC**: `protectedProcedure` vs. `publicProcedure` patterns; middleware via `.use()`.

**Extract**: Complete URL access matrix.

| Path | Method | Public / Authenticated / Role | Notes |
|---|---|---|---|
| `/health` | GET | Public | Health check |
| `/api/users/:id` | GET | Authenticated | Self or admin |
| `/api/admin/*` | ALL | Admin role | |

**Anti-patterns to flag**:
- Auth middleware after the route handler (registered too late).
- Public routes mounted before global auth middleware.
- Inverted ordering of multiple auth checks.

### 3.2 Method-level / Programmatic — P1

Search for explicit role/permission checks in service code:
- `if (!user.isAdmin) throw new ForbiddenError()`
- `@Roles('admin')` decorator (NestJS)
- `ability.can('read', subject)` (CASL library)
- `accesscontrol` library usage

Note **where the user's roles/permissions come from**: JWT claims, DB lookup per request, cached in session.

### 3.3 Role & Permission Model — P1

Map roles, role→permission mapping, and the authority source (JWT claims, DB, external service).

## Step 4: Analyze Security Infrastructure — P1/P2

### 4.1 CORS — P1

- Global config: `cors()` middleware (`cors` package) or framework equivalent.
- Allowed origins: literal list, regex, function returning bool — note which.
- Credentials: `credentials: true` requires explicit origin (not `*`); flag the combination if dangerous.
- Headers exposed/allowed.

### 4.2 Security Headers — P1

| Library | What it sets |
|---|---|
| `helmet` (Express) | CSP, X-Frame-Options, X-Content-Type-Options, Strict-Transport-Security, Referrer-Policy, etc. |
| `@fastify/helmet` | Same for Fastify |
| Manual `res.setHeader(...)` | Custom; check each header |

Note absence of helmet/equivalent — it's the easiest security win and absence is meaningful.

### 4.3 Rate Limiting — P1

- `express-rate-limit` / `@fastify/rate-limit` / `rate-limiter-flexible` / `bottleneck`.
- Per-IP vs. per-user limits.
- Store: in-memory (single instance only) vs. Redis (distributed).
- Endpoints with stricter limits: login, password reset, payment, expensive queries.

### 4.4 Sessions — P1

(See 2.3 above; sessions overlap with auth.)

### 4.5 CSRF — P1

- Stateless API with Bearer auth → CSRF not applicable; no CSRF middleware expected.
- Cookie-based session → CSRF protection IS expected. Look for `csurf` (deprecated but still common), `@fastify/csrf-protection`, or custom token.
- SameSite cookies: `sameSite: 'lax'` or `'strict'` provides partial protection.

### 4.6 Secrets Management — P0

| What to Find | Where to Look |
|---|---|
| Vault | `node-vault`, `@hashicorp/vault-client` |
| AWS Secrets Manager | `@aws-sdk/client-secrets-manager` |
| GCP Secret Manager | `@google-cloud/secret-manager` |
| Doppler | `@dopplerhq/cli` integration via env injection |
| dotenv | `.env` files loaded at startup; secrets in env vars |
| Hardcoded | String literals in source — **flag immediately** |
| Build-time injection | Webpack/Vite `define`, Next.js `NEXT_PUBLIC_*` (note: those are public!) |

**`NEXT_PUBLIC_*` warning**: In Next.js, any env var prefixed `NEXT_PUBLIC_` is bundled into client JS — visible to anyone. Flag if a secret has this prefix.

**Extract**: How secrets are injected, what secrets the app needs, how to configure for local dev.

### 4.7 Custom security middleware — P2

Catalog any custom middleware that touches auth/authz/rate-limiting/header-setting. Trace its order in the middleware chain.

## Step 5: Security Testing — P1

| What to Find | Where to Look |
|---|---|
| Auth helper / mock user | `tests/helpers/auth.ts`, `__mocks__/auth.ts`, sign-test-jwt utility |
| Integration tests for 401/403 | `expect(response.status).toBe(401)` patterns |
| Security-focused suites | `test:security` script, dedicated directory |
| Dependency vulnerability scan | `npm audit` / `pnpm audit` in CI, Snyk, Socket |

## Step 6: Local Development Authentication — P0

| What to Find | Where to Look |
|---|---|
| Dev profile / config | `.env.development`, `.env.local`, NODE_ENV-aware code paths |
| Test users / seed | `prisma/seed.ts`, `scripts/seed-users.ts`, `db:seed` |
| Local IdP | Docker Compose service for Keycloak / Authentik / Hydra / mock IdP |
| Token generation script | Custom script to mint a test JWT, e.g. `scripts/dev-token.ts` |
| HTTP client samples | `.http` / `.rest` files (REST Client extension), Postman collections, Bruno collections |

**Extract**: Step-by-step local auth instructions, test credentials, how to get a valid token.

## Step 7: Generate Security Report

Write using the [security analysis template](../assets/security-analysis-template.md).

**Quick mode**: Fill sections 1 (Security Stack), 2.1 (Auth Mechanism), 5.1 (Secrets — how they're managed), 7 (Local Dev Auth). Mark others `[Not analyzed]`.

Default location: `<output-dir>/security-analysis-report.md` (Standard/Deep only).

## Ledger Update After

Mark area 3.3 complete. Add:
- Auth mechanism and flow
- Authorization model
- Secrets configuration method
- Local dev auth steps
- Security anti-patterns found

## Security Anti-Patterns Checklist

Flag these when found:

- [ ] Hardcoded passwords, API keys, or tokens in source/config
- [ ] `jwt.verify` without `algorithms` array (allows `none`)
- [ ] `cors({ origin: true, credentials: true })` (reflects any origin)
- [ ] `cors({ origin: '*' })` with `credentials: true` (browser will reject but indicates confused config)
- [ ] No rate limiting on `/login`, `/register`, `/password-reset`
- [ ] Session store = in-memory in production code path
- [ ] Cookies without `httpOnly` / `secure` / `sameSite`
- [ ] Passwords stored with SHA-* (not bcrypt/argon2/scrypt)
- [ ] `eval()` or `new Function(...)` taking user input
- [ ] `child_process.exec` with concatenated user input
- [ ] SQL strings interpolating user input (vs. parameterized queries)
- [ ] `req.body` flowing to handlers without validation (Zod/class-validator/Joi/AJV missing)
- [ ] `helmet` (or equivalent) absent in Express/Fastify
- [ ] `NEXT_PUBLIC_*` env var with a secret value
- [ ] `npm audit` errors in `package-lock.json` ignored without justification
- [ ] PII logged in plain text
- [ ] Health endpoint exposed *with* env/config/heap info publicly
- [ ] Prototype-pollution-prone deep-merge libraries unpatched

## Exploration Guidelines

- **Read auth files in full** — `middleware/auth.ts`, `auth.module.ts`, `auth.config.ts`. They define the entire posture.
- **Trace the middleware order** — order is the lifecycle; misordered = bypassed.
- **Check all NODE_ENV branches** — `if (NODE_ENV !== 'production') { auth = false }` is a real pattern.
- **Flag anti-patterns immediately** — don't save for the end.
- **Note absences**: No CORS config? No rate limiting? No security tests? Findings.
- **Don't expose actual secret values** — report locations and variable names, never values.
