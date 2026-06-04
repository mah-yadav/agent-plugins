# Area: Security Analysis

Reference for Phase 3.3 of the onboarding workflow. Produces a structured security report covering authentication, authorization, API security, secrets management, and security testing.

**Core principle**: Understand the security model before you write code. Security misconfigurations are the most common source of vulnerabilities.

## Contents

- Step 1: Identify the Security Stack
- Step 2: Analyze Authentication (2.0 servlet/reactive branch, 2.1 mechanism, 2.2 JWT, 2.3 user details)
- Step 3: Analyze Authorization (3.1 URL-based, 3.2 method-level, 3.3 role model)
- Step 4: Analyze Security Infrastructure (4.1 CORS … 4.6 secrets, 4.7 filters)
- Step 5: Security Testing
- Step 6: Local Development Authentication
- Step 7: Generate Security Report
- Anti-patterns checklist, exploration guidelines

## Priority Tiers (within this area)

| Step | Priority | Rationale |
|---|---|---|
| Step 2: Authentication | **P0** | Must know how auth works to make any API call |
| Step 3.1: URL-Based Authorization | **P0** | Must know which endpoints are public vs. protected |
| Step 4.6: Secrets Management | **P0** | Must know how to configure secrets locally |
| Step 6: Local Dev Auth | **P0** | Must authenticate locally to develop |
| Step 3.2–3.3: Method-Level Security & Role Model | **P1** | Important for writing secured code |
| Step 4.1–4.4: CORS, CSRF, Sessions, Headers | **P1** | Important but not day-1 blockers |
| Step 5: Security Testing | **P1** | Need to write security-aware tests |
| Step 4.5: Rate Limiting | **P2** | Rarely blocks development |
| Step 4.7: Security Filters deep dive | **P2** | Only for security-focused work |

**Quick mode**: Complete Step 1, Step 2.1 (auth mechanism summary), Step 4.6 (secrets), Step 6 (local dev auth). Skip everything else.

## Ledger Read-Before

Check the ledger. If 3.2 (`explain-code`) already identified the security framework, start from that finding. Pull security dependencies from the build file analysis. If `application.yml` was read, reuse the security config properties already found.

Note `Reactive stack?` from Project Identity — see Step 3.0 below; the filter chain bean type and DSL differ on WebFlux.

## Symbol Tracing

To find all usages of `SecurityFilterChain`, `@PreAuthorize`, etc., use `Grep` (precise) or the `Explore` subagent (broader). Read context around hits to disambiguate.

## Step 1: Identify the Security Stack — P0

If the stack was already identified in Phase 3.2, skip to Step 3.

1. **Read the build file** (if not in the ledger) for security dependencies:

   **Spring Boot / Spring Security**:

   | Dependency | Indicates |
   |---|---|
   | `spring-boot-starter-security` | Spring Security core |
   | `spring-security-oauth2-resource-server` | OAuth2 resource server |
   | `spring-security-oauth2-client` | OAuth2 client (login redirect) |
   | `spring-security-oauth2-jose` | JWT/JWS/JWE support |
   | `spring-security-saml2-service-provider` | SAML 2.0 |
   | `jjwt` / `java-jwt` / `nimbus-jose-jwt` | Custom JWT handling |
   | `spring-cloud-starter-vault-config` | HashiCorp Vault |
   | `bucket4j` / `resilience4j-ratelimiter` | Rate limiting |

   **Quarkus**:

   | Dependency | Indicates | Config Prefix |
   |---|---|---|
   | `quarkus-oidc` | OIDC/OAuth2 auth | `quarkus.oidc.*` |
   | `quarkus-smallrye-jwt` | MicroProfile JWT | `mp.jwt.*` |
   | `quarkus-security` | Built-in security | `quarkus.security.*` |
   | `quarkus-keycloak-authorization` | Keycloak policy enforcement | `quarkus.keycloak.*` |

   **Micronaut**:

   | Dependency | Indicates | Config Prefix |
   |---|---|---|
   | `micronaut-security-jwt` | JWT auth | `micronaut.security.token.jwt.*` |
   | `micronaut-security-oauth2` | OAuth2/OIDC | `micronaut.security.oauth2.*` |
   | `micronaut-security-session` | Session-based auth | `micronaut.security.session.*` |

   **Jakarta EE / MicroProfile**:

   | Indicator | Indicates |
   |---|---|
   | `@RolesAllowed`, `@DeclareRoles` on JAX-RS resources | Declarative role-based auth |
   | `web.xml` `<security-constraint>` | URL-based authorization |
   | `@LoginConfig` / `login-config` in `web.xml` | Authentication mechanism (BASIC, FORM, etc.) |
   | MicroProfile JWT (`org.eclipse.microprofile.jwt`) | Token-based auth via MP JWT |

2. **Find security configuration**: Search for `@EnableWebSecurity`, `SecurityFilterChain` (Spring), `@RolesAllowed` (Jakarta EE), `SecurityRule` (Micronaut), or `quarkus.http.auth.*` (Quarkus).
3. **Check application config**: Security properties (`spring.security.*`, `spring.oauth2.*`, `quarkus.oidc.*`, `micronaut.security.*`).

## Step 2: Analyze Authentication — P0

### 2.0 Servlet vs. Reactive Branch (Spring projects)

**Before reading any security config**, determine whether the project uses Spring MVC (servlet) or WebFlux (reactive). Check `Reactive stack?` in the ledger; if absent, look for `spring-boot-starter-webflux` (reactive) vs. `spring-boot-starter-web` (servlet).

| | Servlet (MVC) | Reactive (WebFlux) |
|---|---|---|
| Filter chain bean | `SecurityFilterChain` | `SecurityWebFilterChain` |
| Configurer DSL | `HttpSecurity` | `ServerHttpSecurity` |
| Principal access | `Authentication` parameter, `SecurityContextHolder` | `ReactiveSecurityContextHolder`, `Mono<Principal>` |
| Method security | `@PreAuthorize` on blocking methods | `@PreAuthorize` on methods returning `Mono`/`Flux` |
| Test annotations | `@WithMockUser` | `@WithMockUser` + `WebTestClient.mutateWith(mockUser(...))` |
| Auth manager | `AuthenticationManager` | `ReactiveAuthenticationManager` |

If reactive, grep for `SecurityWebFilterChain` (not `SecurityFilterChain`). Authorization rules use `.authorizeExchange()` instead of `.authorizeHttpRequests()`. The rest of this reference applies — substitute reactive bean types as you read.

### 2.1 Authentication Mechanism — P0

**Spring Security**:

| Mechanism | Detection | Key Analysis |
|---|---|---|
| **OAuth2 Resource Server** | `oauth2ResourceServer()` in filter chain | Token issuer URI, JWT decoder, audience validation |
| **OAuth2 Client (Login)** | `oauth2Login()` in filter chain | Client registration, redirect URIs |
| **JWT (custom)** | `jjwt` dependency, custom filter | Token extraction, signing key, claims |
| **Basic Auth** | `httpBasic()` in filter chain | User store, password encoder |
| **API Key** | Custom filter checking headers | Key storage, validation |
| **SAML 2.0** | `saml2Login()` | IdP metadata, certificates |

**Quarkus**:

| Mechanism | Detection | Key Analysis |
|---|---|---|
| **OIDC / OAuth2** | `quarkus-oidc` dependency | `quarkus.oidc.auth-server-url`, `quarkus.oidc.client-id`, token verification |
| **MicroProfile JWT** | `quarkus-smallrye-jwt` dependency | `mp.jwt.verify.publickey.location`, `mp.jwt.verify.issuer`, claims mapping |
| **Basic Auth** | `quarkus.security.users.*` properties | Embedded users, `elytron-security-properties-file` |
| **Keycloak Policy** | `quarkus-keycloak-authorization` | Policy enforcer config, resource permissions |

**Micronaut**:

| Mechanism | Detection | Key Analysis |
|---|---|---|
| **JWT** | `micronaut-security-jwt` | `micronaut.security.token.jwt.signatures.*`, token generation/validation |
| **OAuth2 / OIDC** | `micronaut-security-oauth2` | `micronaut.security.oauth2.clients.*`, provider config |
| **Session** | `micronaut-security-session` | `micronaut.security.session.*`, login handler |
| **Basic Auth** | `AuthenticationProvider` bean | Custom authentication provider implementation |

**Jakarta EE**:

| Mechanism | Detection | Key Analysis |
|---|---|---|
| **Container Auth** | `<login-config>` in `web.xml` | Auth method (BASIC/FORM/DIGEST/CLIENT-CERT), realm name |
| **MicroProfile JWT** | `org.eclipse.microprofile.jwt` | `mp.jwt.verify.*` properties, `@Claim` injection |
| **Custom Filter** | `@WebFilter` or `ContainerRequestFilter` | Manual token/header extraction |

**Extract**: Authentication mechanism, identity provider, token format, where config lives.

### 2.2 JWT Analysis (if applicable) — P1

| What to Find | Where to Look |
|---|---|
| Token structure | JWT filter or `JwtDecoder` bean — expected claims |
| Signing algorithm | RS256, HS256, ES256 |
| Key management | JWK set URI, local public key, symmetric secret |
| Token validation | Issuer, audience, expiration |
| Token extraction | Authorization header, cookie, query param |

### 2.3 User Details & Credentials — P1

- `UserDetailsService` implementation — where users are loaded from.
- `PasswordEncoder` bean — BCrypt, Argon2, SCrypt.
- User entity, registration flow, password reset.

## Step 3: Analyze Authorization — P0/P1

### 3.1 URL-Based Authorization — P0

Read `SecurityFilterChain` (or `SecurityWebFilterChain` for reactive) and map URL access rules.

**Extract**: Complete URL access matrix — path, HTTP method, required role/authority.

| Path Pattern | HTTP Method | Access Rule | Notes |
|---|---|---|---|
| `/actuator/health` | GET | `permitAll()` | Health check |
| `/api/**` | ALL | `authenticated()` | Requires valid auth |

### 3.2 Method-Level Security — P1

Search for `@PreAuthorize`, `@Secured`, `@RolesAllowed` across the codebase.

**Analyze**: Which methods have security annotations, custom security expressions, role hierarchy, permission evaluator.

### 3.3 Role & Permission Model — P1

Map roles, role-to-permission mapping, role sources (JWT claims, database, LDAP).

## Step 4: Analyze Security Infrastructure — P1/P2

### 4.1 CORS Configuration — P1

Global CORS (`CorsConfigurationSource`), controller-level (`@CrossOrigin`), allowed origins/methods.

### 4.2 CSRF Protection — P1

CSRF status, token repository, ignored paths. Note: `csrf().disable()` with Bearer auth is valid.

### 4.3 Session Management — P1

Session creation policy, session fixation protection, concurrent sessions, session store.

### 4.4 Security Headers — P1

HSTS, X-Frame-Options, X-Content-Type-Options, CSP, Referrer-Policy.

### 4.5 Rate Limiting — P2

Bucket4j, Resilience4j, API Gateway, custom filter, per-client limits.

### 4.6 Secrets Management — P0

| What to Find | Where to Look |
|---|---|
| Vault | `spring-cloud-starter-vault-config`, `spring.cloud.vault.*` |
| AWS Secrets Manager | `aws-secretsmanager` dependency |
| Environment variables | `${SECRET_NAME}` placeholders without defaults |
| Hardcoded secrets | String literals in config or code — **flag immediately** |

**Extract**: How secrets are injected, what secrets the app needs, how to configure for local dev.

### 4.7 Security Filters & Custom Components — P2

Custom filters, filter ordering, authentication providers, entry points, access denied handlers.

## Step 5: Security Testing — P1

| What to Find | Where to Look |
|---|---|
| Mock authentication | `@WithMockUser`, `@WithAnonymousUser` |
| Security test utilities | `SecurityMockMvcRequestPostProcessors` |
| Security integration tests | Tests verifying 401/403 responses |
| Custom test annotations | Meta-annotations with specific roles |

## Step 6: Local Development Authentication — P0

| What to Find | Where to Look |
|---|---|
| Dev profile security | `application-local.yml` / `application-dev.yml` |
| Test users | In-memory users, seed data, dev-only user creation |
| Local IdP | Docker Compose with Keycloak/mock IdP |
| Token generation | Scripts or Makefile targets for test JWTs |
| HTTP client files | `.http` files, Postman collections |

**Extract**: Step-by-step local auth instructions, test credentials, how to get a valid token.

## Step 7: Generate Security Report

Write the report using the [security analysis template](../assets/security-analysis-template.md).

**Quick mode**: Fill sections 1 (Security Stack), 2.1 (Authentication Mechanism), 5.1 (Secrets — how they're managed), 7 (Local Dev Auth). Mark other sections `[Not analyzed — out of scope for Quick mode]`.

**Standard/Deep mode**: Fill all sections.

Default location: `<output-dir>/security-analysis-report.md` (Deep mode only — in Standard mode, security findings roll into the main report).

## Ledger Update After

Mark area 3.3 complete. Add:
- Authentication mechanism and flow
- Authorization model (role-based, permission-based)
- Secrets configuration method
- Local dev auth steps
- Security anti-patterns found

## Security Anti-Patterns Checklist

Flag these when found:

- [ ] Hardcoded passwords, API keys, or tokens in source code or config
- [ ] `csrf().disable()` without justification (stateless API with Bearer auth is valid)
- [ ] `cors().allowedOrigins("*")` with `allowCredentials(true)`
- [ ] `permitAll()` on admin or sensitive endpoints
- [ ] SQL injection via string concatenation in `@Query(nativeQuery = true)`
- [ ] Password stored without `PasswordEncoder`
- [ ] Security disabled for dev profile without safeguard
- [ ] PII logged in plain text
- [ ] Actuator endpoints exposed without authentication in production

## Exploration Guidelines

- **Read security config classes in full** — they define the entire security posture.
- **Follow the filter chain**: Custom filter → trace what it does, where it sits.
- **Check ALL profiles**: Dev may permit all, prod may be locked down.
- **Flag anti-patterns immediately** — don't save for the end.
- **Note absences**: No CORS? No rate limiting? No security tests? Report them.
- **Don't expose actual secrets**: Report locations, never values.
