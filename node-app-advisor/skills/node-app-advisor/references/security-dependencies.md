# Security & Dependencies Analysis

A white-box, code-level security review — not a penetration test. Map findings to OWASP Top 10. Every finding needs evidence, severity, and a concrete fix.

**Core principle:** Security issues found in code review cost 10x less than those found in production.

## Contents

- Step 1 — Security stack identification
- Step 2 — OWASP Top 10 (A01–A10)
- Step 3 — Hardcoded secrets scan
- Step 4 — Dependency health
- Step 5 — Compile findings

## Tools

- `Grep` — security-sensitive patterns: SQL/query strings, crypto calls, secret patterns, auth middleware/guards.
- `Glob` — find security config, keys, `.env*`, dependency files (`package.json`, lockfiles), Dockerfiles, manifests.
- `Read` — read auth middleware/guards, config, and route handlers in full.
- `Bash` — read-only dependency commands: `npm audit --omit=dev`, `npm outdated`, `npm ls <pkg>`, `pnpm audit`, `yarn audit` (Yarn Classic v1) / `yarn npm audit` (Yarn Berry v2+).
- `Agent` (Explore) — fan-out: "find all raw SQL query strings", "find every `child_process` call". Re-`Read` leads before citing.
- **Usages:** `Grep` the export/middleware name across the tree (no `listCodeUsages` tool). Mind tsconfig `paths` aliases.

## Step 1 — Security stack identification

1. **Security-related dependencies** (read `package.json`):

   | Dependency | Concern |
   |---|---|
   | `helmet` | HTTP security headers — verify it's actually applied and configured |
   | `cors` | CORS — check origin allowlist vs. `*` |
   | `express-rate-limit` / `@fastify/rate-limit` / `rate-limiter-flexible` | Rate limiting — coverage of auth endpoints |
   | `passport` / `passport-*` | Auth strategies — session vs. token |
   | `jsonwebtoken` / `jose` / `@fastify/jwt` | JWT — signing alg, verification, expiry |
   | `bcrypt` / `bcryptjs` / `argon2` / `scrypt` | Password hashing — algorithm & cost |
   | `csurf` / `@fastify/csrf-protection` | CSRF — needed for cookie/session apps |
   | `express-session` / `@fastify/session` / `cookie-session` | Session — cookie flags, store |
   | `zod` / `joi` / `class-validator` / `yup` / `express-validator` | Input validation — applied at boundaries? |
   | `dotenv` | Env loading — secrets not committed |
   | `@nestjs/passport` / `@nestjs/jwt` / `@nestjs/throttler` | NestJS security, JWT, rate limiting |

2. **Security configuration by framework:**

   | Framework | Detection |
   |---|---|
   | Express | `app.use(helmet())`, `app.use(cors(...))`, auth middleware order, `express-rate-limit` |
   | Fastify | `fastify.register(helmet)`, `@fastify/cors`, `@fastify/jwt`, `preHandler` auth hooks |
   | NestJS | `@UseGuards(...)`, `AuthGuard`, `@Roles()`, global guards in `main.ts`, `ValidationPipe` |
   | Koa | `koa-helmet`, `@koa/cors`, auth middleware in the chain |
   | Hono | `hono/csrf`, `hono/cors`, `hono/jwt`, `bearerAuth` middleware |
   | Next.js | `middleware.ts` auth checks, route-handler session validation, `headers()` config |

3. **Auth mechanism** — OAuth2/OIDC, JWT (and where it's stored — cookie vs. localStorage), session cookies, API key, Basic, or custom.

## Step 2 — OWASP Top 10 (2021)

### A01 — Broken Access Control

| Vulnerability | Detection | Severity |
|---------------|-----------|----------|
| Missing route-level authorization | Data-modifying routes without an auth middleware/guard/role check | Critical |
| IDOR | Endpoints taking resource IDs without ownership validation (`/users/:id` not checking the current user) | Critical |
| Missing/permissive CORS | `cors({ origin: '*', credentials: true })` or reflecting any `Origin` | High |
| Privilege escalation | Admin-only routes reachable without a role/permission check | Critical |
| Missing CSRF protection | Cookie/session app with state-changing routes and no CSRF token | High |
| Path traversal | User input in `fs`/`path.join`/`sendFile` without normalization & containment checks | Critical |
| Insecure default-allow | Global middleware allowing all, with auth opt-in per route (easy to forget) | High |
| Mass assignment | Spreading `req.body` straight into an ORM `create`/`update` without field allowlist | High |

*Framework specifics:* Express — confirm auth middleware runs **before** the handler and isn't skipped by route order. NestJS — global vs. per-controller guards; `@Public()` decorators bypassing global auth. Fastify — `preHandler`/`onRequest` auth hooks. Next.js — `middleware.ts` matcher coverage; per-route-handler checks (middleware alone is not authorization).

### A02 — Cryptographic Failures

| Vulnerability | Detection | Severity |
|---------------|-----------|----------|
| Weak hashing | MD5/SHA-1 (`crypto.createHash('md5'\|'sha1')`) for passwords or integrity | Critical |
| Fast hash for passwords | Plain SHA-256 for passwords instead of bcrypt/argon2/scrypt | Critical |
| Hardcoded encryption keys/secrets | Keys, JWT secrets in source or committed config | Critical |
| Insecure random | `Math.random()` for tokens/IDs/keys instead of `crypto.randomBytes`/`randomUUID` | High |
| Weak/legacy ciphers | `crypto.createCipher` (deprecated, no IV), DES, ECB mode | High |
| Missing encryption at rest | PII/credentials stored plaintext | High |
| Weak TLS / disabled verification | `rejectUnauthorized: false`, `NODE_TLS_REJECT_UNAUTHORIZED=0` | High |

*Detect:* grep `createHash('md5'`, `createHash('sha1'`, `Math.random()` near token/id/key, `createCipher(` (vs `createCipheriv`), `rejectUnauthorized: false`, hardcoded strings near `jwt.sign`/`createCipheriv`.

### A03 — Injection

| Vulnerability | Detection | Severity |
|---------------|-----------|----------|
| SQL Injection | String-concatenated / template-literal SQL; `query(\`... ${userInput}\`)`; raw queries without parameter binding | Critical |
| ORM raw-query injection | Prisma `$queryRawUnsafe`/`$executeRawUnsafe`, TypeORM `query()` with interpolation, Sequelize `replacements` bypassed | Critical |
| NoSQL Injection | Unsanitized objects into Mongo queries (`{ $where: ... }`, operator injection via `req.body`) | Critical |
| Command Injection | `child_process.exec`/`execSync` with user input (vs `execFile`/`spawn` with arg array) | Critical |
| Prototype pollution | Merging untrusted objects (`__proto__`/`constructor`) via lodash `merge`, `Object.assign`, custom deep-merge | High |
| XSS | User input rendered without escaping in templates/HTML; `dangerouslySetInnerHTML`; `res.send(userInput)` as HTML | High |
| Log Injection | User input logged unsanitized (CRLF / forged log entries) | Medium |
| ReDoS | User input matched against a catastrophic-backtracking regex, or user-supplied regex | High |
| SSRF via redirects | `res.redirect(userInput)` / open redirect | High |

*Detect:* grep `\$queryRawUnsafe`/`\$executeRawUnsafe`, `.query(` with `` ` `` template literals, `exec(`/`execSync(`, `eval(`, `Function(`, `dangerouslySetInnerHTML`, lodash `merge(`; check that route inputs pass through Zod/Joi/class-validator before use.

### A04 — Insecure Design

| Vulnerability | Detection | Severity |
|---------------|-----------|----------|
| Missing rate limiting | Login/password-reset/OTP without rate limiting | High |
| Missing input validation | Request bodies/params/query consumed without a schema validator | High |
| Missing body-size limits | No `express.json({ limit })` / Fastify `bodyLimit` — DoS via huge payloads | Medium |
| Enumeration attacks | Different responses/timing for "user not found" vs "wrong password" | Medium |
| Missing account lockout | No lockout/backoff after repeated failed logins | High |
| Insecure password reset | Predictable tokens, no expiry, reusable | Critical |

### A05 — Security Misconfiguration

| Vulnerability | Detection | Severity |
|---------------|-----------|----------|
| Missing security headers | No `helmet`/equivalent on a browser-facing app | Medium |
| Debug/verbose errors in prod | Stack traces returned to clients; `NODE_ENV` not set to `production`; source maps served | High |
| Default credentials | Default admin/seed passwords, default DB creds | Critical |
| Verbose error responses | Sending `err.stack`/internal messages in responses | Medium |
| Exposed dev tooling | GraphQL playground / Swagger UI / `/debug` reachable in prod | Medium |
| Permissive CORS | `origin: '*'` with credentials (also A01) | High |
| Trusting `X-Forwarded-*` | `app.set('trust proxy', true)` blindly, enabling IP spoofing for rate limits | Medium |
| Directory listing / static exposure | Serving `.env`, `.git`, source via static middleware | High |

*Detect:* read the app bootstrap for `helmet`/error-handler setup; grep `NODE_ENV`; check the global error handler doesn't leak `stack`; grep `trust proxy`; check static-file roots.

### A06 — Vulnerable & Outdated Components

| Vulnerability | Detection | Severity |
|---------------|-----------|----------|
| Known CVE in dependencies | `npm audit` advisories | High (Critical if known-exploited or CVSS ≥ 9) |
| EOL runtime / framework | Node version below current LTS; framework major past EOL | High |
| Abandoned dependencies | No releases in > 2 years, unmaintained | Medium |
| License risks | Copyleft (GPL/AGPL) deps in a proprietary codebase | Medium |
| Duplicate/conflicting versions | Multiple major versions of the same lib in the lockfile | Medium |
| Lockfile drift / missing lockfile | No committed lockfile, or lockfile out of sync with `package.json` | Medium |

*Detect:* `npm audit --omit=dev` (or `pnpm audit` / `yarn audit` for Yarn v1, `yarn npm audit` for Yarn Berry); `npm outdated`; check `engines.node` vs. current LTS; check for a committed lockfile; `npm ls <pkg>` for duplicate versions.

*Confirming CVEs (opt-in):* `npm audit` already maps to the GitHub Advisory DB and is authoritative — prefer it over guessing. For a **handful of critical/high findings** where you want detail, you may confirm against published advisories with `WebSearch`/`WebFetch` (e.g. `"<pkg> <version> CVE"`, the GitHub Advisory / NVD page) — but **ask the user first** (it leaves the codebase and sends dependency coordinates to an external service). Don't bulk-search every dependency.

### A07 — Identification & Authentication Failures

| Vulnerability | Detection | Severity |
|---------------|-----------|----------|
| Weak password policy | No complexity/min-length enforcement | High |
| Missing MFA | No MFA for admin/privileged operations | Medium |
| Insecure session cookies | Missing `httpOnly`/`secure`/`sameSite` on session/auth cookies | High |
| Session not regenerated | No session-ID regeneration after login (fixation) | High |
| JWT in localStorage | Tokens stored in `localStorage` (XSS-exfiltratable) instead of httpOnly cookies | Medium |
| Weak JWT config | `alg: none` accepted, no `exp`, excessive expiry, secret reused across envs, not verifying `aud`/`iss` | High |
| Long-lived tokens, no revocation | No refresh/rotation/blacklist strategy | Medium |

*Detect:* read cookie options in `express-session`/`cookie` calls; read `jwt.sign`/`jwt.verify` options (algorithms allowlist, `expiresIn`); grep `localStorage` in any served client code.

### A08 — Software & Data Integrity Failures

| Vulnerability | Detection | Severity |
|---------------|-----------|----------|
| Insecure deserialization | `eval`/`Function`/`vm` on untrusted input; `node-serialize`/unsafe YAML load | Critical |
| Prototype pollution sink | Untrusted keys into object merges (also A03) | High |
| `npm install` lifecycle risk | Postinstall scripts from untrusted deps; no `--ignore-scripts` policy | Medium |
| Missing lockfile integrity | No lockfile / not using `npm ci` in CI | Low |
| Unverified dynamic code load | `require(userInput)` / dynamic `import(userInput)` | High |

### A09 — Security Logging & Monitoring Failures

| Vulnerability | Detection | Severity |
|---------------|-----------|----------|
| No audit logging | Auth events, access decisions, data mutations unlogged | High |
| Sensitive data in logs | Passwords/tokens/PII/card numbers logged (also full `req.body`/`req.headers`) | Critical |
| No security event alerting | No alerting on failed-login spikes | Medium |
| Insufficient log context | Security events without user ID, IP, timestamp, request ID | Medium |

*Detect:* read logging in auth code; grep `console.log`/`logger.info` near `password`/`token`/`req.body`/`authorization`; check for an audit-log mechanism.

### A10 — Server-Side Request Forgery

| Vulnerability | Detection | Severity |
|---------------|-----------|----------|
| URL from user input | `fetch`/`axios`/`got`/`undici` with a user-supplied URL and no allowlist | Critical |
| Redirect from user input | `res.redirect(userInput)` (open redirect) | High |
| Internal service access | No allowlist/blocklist (and no blocking of `localhost`/link-local/metadata `169.254.169.254`) for outbound requests | Medium |
| Webhook/callback URLs unvalidated | Storing & calling user-provided callback URLs | High |

## Step 3 — Hardcoded secrets scan

| Pattern | Regex (grep) |
|---------|--------------|
| API keys | `api[_-]?key\s*[:=]\s*['"\`][A-Za-z0-9]` |
| Passwords | `password\s*[:=]\s*['"\`][^$\{]` (not a `process.env`/`${...}` placeholder) |
| JWT/signing secrets | `secret\s*[:=]\s*['"\`][A-Za-z0-9]`, `jwtSecret`, `signingKey` |
| Connection strings | `postgres(ql)?://.*:.*@`, `mongodb(\+srv)?://.*:.*@`, `mysql://.*:.*@`, `redis://.*:.*@` |
| AWS credentials | `AKIA[0-9A-Z]{16}`, `aws_secret_access_key` |
| Private keys | `BEGIN RSA PRIVATE KEY`, `BEGIN PRIVATE KEY`, `BEGIN OPENSSH PRIVATE KEY` |
| Provider tokens | `sk_live_`, `ghp_`, `xox[baprs]-`, `AIza[0-9A-Za-z\-_]{35}` |

Check all config (`*.config.{js,ts}`, `config/**`), source near security operations, committed `.env*` files (should be gitignored), Dockerfiles/Compose/Kubernetes manifests, and confirm `.gitignore` excludes `.env*`. **Critically: check git history isn't shipping a committed `.env`** — `git log --all --full-history -- '**/.env*' '.env*'` (any hits mean a secret was committed even if later removed; skip gracefully if not a git repo).

## Step 4 — Dependency health

1. **Vulnerabilities** — `npm audit` (or pnpm/yarn equivalent); triage by severity and whether the path is reachable. 2. **Node version** — `engines.node` vs. current LTS; flag EOL. 3. **Framework currency** — major version vs. its supported line. 4. **Abandoned deps** — no releases in > 2 years; look for known-deprecated packages (`request`, `node-sass`, `moment` for new code). 5. **Lockfile** — committed and used with `npm ci` in CI. 6. **Duplicate versions** — multiple majors of the same lib bloating the bundle / causing instanceof bugs.

## Step 5 — Compile findings

For each finding: **OWASP category** (A01–A10) · **Severity** · **Description** · **Evidence** (`file:line` + snippet) · **Impact** (what an attacker could do / business risk) · **Recommended fix** (before/after) · **Effort** (XS/S/M/L/XL). Present the compiled findings.
