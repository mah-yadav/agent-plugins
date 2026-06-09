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

- `Grep` — security-sensitive patterns: SQL strings, crypto calls, secret patterns, security annotations.
- `Glob` — find security config, keystores, `.env`, dependency files, manifests.
- `Read` — read security config, filter chains, and controllers in full.
- `Bash` — read-only dependency commands: `mvn dependency:tree`, `mvn versions:display-dependency-updates`, `gradle dependencies`.
- `Agent` (Explore) — fan-out: "find all SQL query strings", "find every `@Query` annotation". Re-`Read` leads before citing.
- **Usages:** `Grep` the class/method name across the tree (no `listCodeUsages` tool).

## Step 1 — Security stack identification

1. **Security-related dependencies** (read the build file):

   | Dependency | Concern |
   |---|---|
   | `spring-boot-starter-security` | Spring Security — analyze config |
   | `spring-security-oauth2-*` | OAuth2 — token validation |
   | `jjwt` / `java-jwt` / `nimbus-jose-jwt` | JWT — signing, validation |
   | `bcrypt` / `argon2` / `scrypt` | Password hashing — algorithm choice |
   | `jasypt` | Property encryption — key management |
   | `bouncy-castle` | Crypto — algorithm choices |
   | `spring-boot-starter-actuator` | Actuator — endpoint exposure |
   | `springdoc-openapi` / `swagger` | API docs — production exposure |
   | `io.micronaut.security:*` | Micronaut Security / JWT / OAuth2 |
   | `io.quarkus:quarkus-security` / `quarkus-oidc` / `quarkus-smallrye-jwt` / `quarkus-keycloak-authorization` | Quarkus security, OIDC, MP-JWT, policy enforcement |

2. **Security configuration by framework:**

   | Framework | Detection |
   |---|---|
   | Spring Boot | `@EnableWebSecurity`, `SecurityFilterChain`, `WebSecurityConfigurerAdapter`, `spring.security.*` |
   | Micronaut | `@Secured`, `micronaut.security.*` in `application.yml`, `SecurityRule` implementations |
   | Quarkus | `@RolesAllowed`/`@PermitAll`/`@DenyAll`, `quarkus.http.auth.*` / `quarkus.oidc.*` |
   | Plain Java | Servlet filters, `web.xml` security constraints, JAAS, custom auth interceptors |

3. **Auth mechanism** — OAuth2, JWT, Basic, Form Login, API Key, mTLS, or custom.

## Step 2 — OWASP Top 10 (2021)

### A01 — Broken Access Control

| Vulnerability | Detection | Severity |
|---------------|-----------|----------|
| Missing method-level authorization | Data-modifying methods without `@PreAuthorize`/`@Secured`/role checks | Critical |
| IDOR | Endpoints taking user IDs without ownership validation (`/api/users/{id}` not checking current user) | Critical |
| Missing CORS restrictions | `allowedOrigins("*")` + `allowCredentials(true)` | High |
| Privilege escalation | Admin-only endpoints reachable without role checks | Critical |
| Missing CSRF protection | `csrf().disable()` on a session-based (non-API) app | High |
| Path traversal | User input in file paths without sanitization (`new File(userInput)`) | Critical |
| Insecure default-deny | Filter chain ending `anyRequest().permitAll()` instead of `.authenticated()` | High |

*Framework specifics:* Spring — read `SecurityFilterChain`, check the final rule; grep `@PreAuthorize`/`@Secured`. Micronaut — `micronaut.security.intercept-url-map`, `@Secured`, `reject-not-found=true`. Quarkus — `quarkus.http.auth.permission`, `@RolesAllowed`/`@PermitAll`/`@DenyAll`. Plain Java — `web.xml` `<security-constraint>`, servlet filters.

### A02 — Cryptographic Failures

| Vulnerability | Detection | Severity |
|---------------|-----------|----------|
| Weak hashing | MD5/SHA-1 for passwords or integrity | Critical |
| Hardcoded encryption keys | Keys in source, properties, or constants | Critical |
| Weak TLS | TLS 1.0/1.1, weak cipher suites | High |
| Missing encryption at rest | PII/credentials stored plaintext | High |
| Insecure random | `java.util.Random` for tokens/keys instead of `SecureRandom` | High |
| ECB mode | AES in ECB instead of GCM/CBC | High |

*Detect:* grep `MessageDigest.getInstance("MD5"|"SHA-1")`, `new Random()` near token/key generation, hardcoded strings near `SecretKeySpec`/`encrypt`/`decrypt`; read `server.ssl.*`.

### A03 — Injection

| Vulnerability | Detection | Severity |
|---------------|-----------|----------|
| SQL Injection | String concatenation in SQL, `@Query` with `+` | Critical |
| JPQL/HQL Injection | Dynamic JPQL/HQL via string concatenation | Critical |
| NoSQL Injection | Unsanitized input in Mongo/Elasticsearch queries | Critical |
| LDAP Injection | Unsanitized input in LDAP filters | Critical |
| Command Injection | `Runtime.exec()` / `ProcessBuilder` with user input | Critical |
| Log Injection | User input logged unsanitized (Log4Shell vector) | High |
| XSS | User input rendered without encoding | High |
| Template Injection | User input in Thymeleaf/Freemarker expressions | Critical |

*Detect:* grep `"SELECT.*" +`, `@Query.*\+`, `Runtime.getRuntime().exec(`, `ProcessBuilder(`, `@Query(nativeQuery = true)`; check for `@Valid`/`@Validated` on controller params.

### A04 — Insecure Design

| Vulnerability | Detection | Severity |
|---------------|-----------|----------|
| Missing rate limiting | Login/password-reset/OTP without rate limiting | High |
| Missing input validation | Request bodies without `@Valid`/`@Validated` | High |
| Enumeration attacks | Different messages for "user not found" vs "wrong password" | Medium |
| Missing account lockout | No lockout after repeated failed logins | High |
| Insecure password reset | Predictable tokens, no expiration, reusable | Critical |

### A05 — Security Misconfiguration

| Vulnerability | Detection | Severity |
|---------------|-----------|----------|
| Debug mode in production | `debug=true`, `spring.jpa.show-sql=true`, verbose errors in non-dev profiles | High |
| Default credentials | Default admin passwords, H2 console with defaults | Critical |
| Exposed actuator endpoints | `/actuator/env`/`/beans`/`/heapdump` unsecured | Critical |
| Exposed API docs | Swagger/OpenAPI UI reachable in production | Medium |
| Stack traces in responses | `server.error.include-stacktrace=always` | Medium |
| Unnecessary HTTP methods | PUT/DELETE/PATCH enabled where not needed | Low |
| Missing security headers | No CSP, X-Frame-Options, X-Content-Type-Options | Medium |

*Detect:* read `application*.yml`; grep `management.endpoints.web.exposure.include=*`, `spring.h2.console.enabled=true`; check for `@ControllerAdvice`/`@ExceptionHandler`.

### A06 — Vulnerable & Outdated Components

| Vulnerability | Detection | Severity |
|---------------|-----------|----------|
| Known CVE in dependencies | Dependencies with published CVEs | High (Critical if known-exploited or CVSS ≥ 9) |
| EOL framework versions | Spring Boot below its current supported line; Java below the current LTS | High |
| Abandoned dependencies | No releases in > 2 years | Medium |
| License risks | GPL-licensed deps in a proprietary codebase | Medium |
| Version conflicts | Multiple versions of the same library on the classpath | Medium |

*Detect:* `mvn dependency:tree` / `gradle dependencies`; `mvn versions:display-dependency-updates`; check Spring Boot version vs. supported releases; check Java version (`maven.compiler.source` / `sourceCompatibility`); check for a BOM / `<dependencyManagement>`.

*Confirming CVEs (opt-in):* findings here are version/EOL heuristics, not live CVE matches. For a **handful of critical or high dependency findings**, you may confirm against published advisories with `WebSearch`/`WebFetch` (e.g. search `"<group:artifact> <version> CVE"` or the GitHub Advisory / NVD pages) — but **ask the user first** (it leaves the codebase and sends dependency coordinates to an external service), and still recommend an SCA tool (`mvn org.owasp:dependency-check:check`, Snyk, GitHub Dependabot) for authoritative results. Do not bulk-search every dependency.

### A07 — Identification & Authentication Failures

| Vulnerability | Detection | Severity |
|---------------|-----------|----------|
| Weak password policy | No complexity/min-length enforcement | High |
| Missing MFA | No MFA for admin/privileged operations | Medium |
| Session fixation | No session-ID regeneration after login | High |
| Insecure session config | Missing `httpOnly`/`secure`/`sameSite` on session cookies | High |
| Token in localStorage | JWT/tokens in browser localStorage (XSS-accessible) | Medium |
| No token expiration | JWT without `exp` or excessively long expiry | High |

### A08 — Software & Data Integrity Failures

| Vulnerability | Detection | Severity |
|---------------|-----------|----------|
| Insecure deserialization | `ObjectInputStream.readObject()` on untrusted data, Jackson polymorphic deserialization | Critical |
| Missing dependency verification | No checksum verification in build | Low |
| Unsigned artifacts | No signature verification for plugins/dependencies | Low |
| Auto-update without verification | Dynamic code loading without integrity checks | High |

### A09 — Security Logging & Monitoring Failures

| Vulnerability | Detection | Severity |
|---------------|-----------|----------|
| No audit logging | Auth events, access decisions, data mutations unlogged | High |
| Sensitive data in logs | Passwords/tokens/PII logged plaintext | Critical |
| No security event alerting | No SIEM/alerting on failed logins | Medium |
| Insufficient log context | Security events without user ID, IP, timestamp | Medium |

*Detect:* read logging in security code; grep `password`/`token`/`secret` near `log.`; check audit mechanisms (`@EntityListeners`, Envers, custom interceptors).

### A10 — Server-Side Request Forgery

| Vulnerability | Detection | Severity |
|---------------|-----------|----------|
| URL from user input | `RestTemplate`/`WebClient`/`HttpClient` with user-supplied URLs | Critical |
| Redirect from user input | `response.sendRedirect(userInput)` | High |
| Internal service access | No allowlist/blocklist for outbound requests | Medium |

## Step 3 — Hardcoded secrets scan

| Pattern | Regex (grep) |
|---------|--------------|
| API keys | `api[_-]?key\s*=\s*['"][A-Za-z0-9]` |
| Passwords | `password\s*=\s*['"][^$\{]` (not a `${...}` placeholder) |
| JWT secrets | `secret\s*=\s*['"][A-Za-z0-9]`, `signing-key\s*=` |
| Connection strings | `jdbc:.*password=`, `mongodb://.*:.*@` |
| AWS credentials | `AKIA[0-9A-Z]{16}`, `aws_secret_access_key` |
| Private keys | `BEGIN RSA PRIVATE KEY`, `BEGIN PRIVATE KEY` |

Check all `application*.yml`/`*.properties`, Java source near security operations, `.env`/Docker Compose/Kubernetes manifests, and confirm `.gitignore` excludes sensitive files.

## Step 4 — Dependency health

1. **Version currency** vs. latest stable. 2. **Spring Boot** vs. its currently supported line. 3. **Java** — flag any release below the current LTS; recommend an upgrade path. 4. **Dependency management** — BOM present, versions centralized? 5. **Transitive risks** — convergence issues. 6. **Abandoned deps** — no releases in > 2 years.

## Step 5 — Compile findings

For each finding: **OWASP category** (A01–A10) · **Severity** · **Description** · **Evidence** (`file:line` + snippet) · **Impact** (what an attacker could do / business risk) · **Recommended fix** (before/after) · **Effort** (XS/S/M/L/XL). Present the compiled findings.
