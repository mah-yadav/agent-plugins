# Node Web Frameworks: request-handling detection & analysis

How to detect and read the framework that handles HTTP requests. Load this for
routing, middleware, controllers, request handlers, "where do routes live",
"what does this guard/decorator do". For GraphQL, gRPC, WebSockets, and message
queues, see **integrations.md** instead. For the per-framework *request lifecycle
order*, see **control-flow.md**.

**Contents** — Express · Fastify · NestJS · Koa · Hapi · Hono · tRPC · Next.js (API routes)

---

## Express (most common server framework)

**Detection**: `express` in dependencies. Look for `app = express()` or `const router = express.Router()`.

**Key patterns**:
- **Routes**: `app.get('/path', handler)`, `app.post(...)`, or `router.use('/api/v1', subRouter)`.
- **Middleware order matters**: Express runs middleware in registration order. `app.use(cors())` before `app.use(bodyParser.json())` etc.
- **Error handling**: Express recognizes middleware with 4 args `(err, req, res, next)` as an error handler. Must be registered LAST.
- **Async errors**: Express 4 does NOT auto-catch errors from async handlers. Look for `express-async-errors`, `asyncHandler`, or manual `try/catch + next(err)`. Express 5 auto-catches.
- **Common middleware**: `helmet`, `cors`, `morgan`, `compression`, `express-rate-limit`, `cookie-parser`, `csurf`, `express-session`, `passport`, `multer` (file uploads).

**Analysis checklist**:
1. Map all routes — search for `\.(get|post|put|patch|delete|all|use)\(` patterns.
2. Identify the middleware stack — order is the request lifecycle.
3. Find the error handler — verify it has 4 args.
4. Check for async-handler wrapping pattern.
5. Look at how `req.user` (or similar) gets populated — that's the auth boundary.

## Fastify

**Detection**: `fastify` in dependencies. `const fastify = Fastify({...})` at bootstrap.

**Key differences from Express**:
- Schema-based: Routes can declare request/response JSON Schema for validation + serialization.
- Plugins: Fastify uses an encapsulation model (`fastify.register(plugin, opts)`). Plugin scope matters.
- Hooks: `onRequest`, `preHandler`, `onResponse`, `onError`, `preValidation`, `preSerialization` — these are the lifecycle.
- Better async handling natively.

**Analysis checklist**:
- Routes: `fastify.get('/path', { schema: {...} }, handler)`. Search for `.route(`, `.get(`, `.post(`.
- Plugins: `fastify-plugin`, `@fastify/cors`, `@fastify/jwt`, `@fastify/swagger`, `@fastify/multipart`.
- Schemas: type-providers (`@fastify/type-provider-typebox`, `@fastify/type-provider-json-schema-to-ts`).

## NestJS (Angular-like, decorator-heavy)

**Detection**: `@nestjs/core`, `@nestjs/common` in dependencies. `@Module`, `@Controller`, `@Injectable` decorators. `main.ts` calls `NestFactory.create(AppModule)`.

**This is the framework most similar to Spring Boot in spirit**: heavy DI, decorator-driven, opinionated module structure.

**Key annotations**:

| Decorator | Layer | Purpose |
|---|---|---|
| `@Module` | Bootstrap | Declares a module's controllers, providers, imports, exports |
| `@Controller('path')` | Web | Group of routes under a path |
| `@Get('/sub')`, `@Post`, `@Put`, `@Delete`, `@Patch` | Web | HTTP route on a controller |
| `@Injectable()` | Service / Repo | DI-managed class |
| `@Inject(TOKEN)` | DI | Inject by token (vs. by type) |
| `@Body()`, `@Param()`, `@Query()`, `@Headers()` | Web | Argument decorators |
| `@UseGuards(JwtAuthGuard)` | Web | Auth check for route/controller |
| `@UseInterceptors()` | Web | Transform request/response |
| `@UsePipes(ValidationPipe)` | Web | Validate input |
| `@CacheKey`, `@CacheTTL` | Cache | From `@nestjs/cache-manager` |

**Important — decorator-generated behavior is invisible in source**: `@Controller`, `@Get`, `@Injectable`, and many others register metadata at module-init time. The visible source doesn't show *how* routes are wired — NestJS reflects on metadata. If you see decorators, the wiring is implicit. (See **foundation.md** § Decorator-aware reading.)

**NestJS modules**: An import graph between `@Module`-decorated classes. Read `AppModule` first; trace its `imports:` array.

**HTTP under the hood**: NestJS sits on Express *or* Fastify (chosen at `NestFactory.create()`). For analysis, the framework-specific middleware applies underneath.

## Koa

**Detection**: `koa` package. `const app = new Koa()`.

- Middleware is a chain of async functions: `async (ctx, next) => { ...; await next(); ... }`.
- The `ctx` object holds request, response, state.
- Used by Strapi (CMS) and Egg.js.

## Hapi

**Detection**: `@hapi/hapi`. `const server = Hapi.server({...})`.

- Schema-first via `joi` (Hapi's sibling library).
- Route options object includes auth strategies, validation, response schemas.
- Less common today but still in production.

## Hono

**Detection**: `hono` in dependencies.

- Edge-runtime-first (Cloudflare Workers, Vercel Edge, Deno, Node).
- Express-like API but **request handlers return `Response` objects** (Web standard).
- Often paired with `@hono/zod-validator` for input validation.
- If you see Hono + `wrangler.toml`, treat as Workers app — Node-specific APIs may not work.

## tRPC

**Detection**: `@trpc/server` in dependencies.

- Procedure-based, not REST-based. Functions defined in routers, called over HTTP under the hood.
- Schema-first via Zod (input + output validation).
- Type-safe client-server contract — frontend imports the *types* (not runtime code) of the backend.
- Routers compose: `router({ user: userRouter, post: postRouter })`.
- Middleware via `t.middleware(...)` chained with `.use()`.

## Next.js (API routes only — backend portion)

**Detection**: `next` in dependencies, `pages/api/` or `app/api/` directories.

- **Pages Router** (`pages/api/foo.ts`): default export is the handler `(req, res) => ...`.
- **App Router** (`app/api/foo/route.ts`): named exports `GET`, `POST`, etc., returning `Response`.
- Edge runtime: `export const runtime = 'edge'` at the top of an API file — restricted runtime.
- Middleware: `middleware.ts` at project root or in `app/` directory.
- `next.config.{js,ts}`: rewrites, headers, image domains, runtime config.

This guide focuses on **backend Node**. Next.js frontend (React) is out of scope here — note its presence but recommend a separate analysis.
