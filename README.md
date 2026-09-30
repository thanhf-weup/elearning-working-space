# WEUP E-Learning — Workspace Guide

This directory is a **workspace, not a repository**. It holds five independent git repos that
together make up the WEUP e-learning platform (weupbook.com). There is no root `package.json`,
no workspaces field, no shared types package, and no root `.git` — each sub-directory is its own
repo with its own branch, lockfile and release pipeline.

That fact shapes everything else in this document. Because the repos are separate, a feature that
touches both server and client **must be built backend-first, then one client at a time**. You
cannot hold a single editing session (or a single AI agent) across the boundary, and you cannot
merge a client change that depends on an endpoint which has not shipped yet.

> **New here? Read [`docs/README.md`](docs/README.md) next.** It contains the ordered, checkable
> procedure for shipping a feature across all five repos.

---

## 1. The five projects

| Directory | Stack | Role | Dev port |
|---|---|---|---|
| [`elearning-authentication-service`](elearning-authentication-service/) | **Fastify 5** + TypeScript + Drizzle | Credentials, Google OAuth, OTP, password reset, RBAC tables. **The only service that issues JWTs.** | `8000` |
| [`elearning-core-service`](elearning-core-service/) | **NestJS 11** (hexagonal) + Drizzle + BullMQ | All product domain: catalog, courses, ebooks, audiobooks, podcasts, orders, cart, blog, recruitment, POS. | `3435` |
| [`elearning-cms`](elearning-cms/) | **Vite + React 19 + Ant Design 6** (SPA) | Internal back-office for admins, content editors and operators. | `4000` |
| [`elearning-fe`](elearning-fe/) | **Turborepo + Next.js 16** (App Router) | Consumer storefront `apps/web`; B2B partner portal `apps/workspace` (early scaffold). | `3000` / `3001` |
| [`elearning-mobile-app`](elearning-mobile-app/) | **Flutter 3.41 + Riverpod 3** | iOS + Android learner app. Web is explicitly out of scope. | — |

Sizes, so you know what you are walking into. Re-derive any of these with the commands in
[`docs/09-local-setup.md`](docs/09-local-setup.md#appendix-re-deriving-the-counts) rather than
trusting this table after a few months:

| | core-service | auth-service | cms | fe (`apps/web`) | mobile |
|---|---|---|---|---|---|
| Feature units | 275 use cases | 29 routes | 16 modules | 26 modules | 19 features |
| HTTP surface | 50 controllers | 2 controllers | 20 services | 22 services | 1 endpoint registry |
| Data | 68 tables, 172 migrations | 7 tables, 11 migrations | — | — | — |
| Tests | 137 Jest specs | 2 Vitest files | **none** | **none** | full suite + coverage gate |
| Locales | 8 | 2 | **none** | 9 | 9 |

Every project ships its own package manager lockfile. All four Node projects use **npm**. Node
versions disagree between local and CI — see §8.

---

## 2. How a request flows

```
 ┌───────────┐  ┌───────────┐  ┌───────────┐  ┌────────────┐
 │    CMS    │  │    Web    │  │  Mobile   │  │ Workspace  │
 │  (React)  │  │ (Next.js) │  │ (Flutter) │  │  (Next.js) │
 └─────┬─────┘  └─────┬─────┘  └─────┬─────┘  └─────┬──────┘
       │              │              │              │
       └──────────────┴──────┬───────┴──────────────┘
                             │  one host, e.g. staging.api.weupbook.com
                    ┌────────▼────────┐
                    │  Kong  gateway  │   ← the security boundary (config in neither repo)
                    └────────┬────────┘
              /api/v1/auth/* │ /api/v1/coreservice/*
              ┌──────────────┴───────────────┐
              ▼                              ▼
   ┌─────────────────────┐        ┌─────────────────────┐
   │   auth-service      │        │    core-service     │
   │     (Fastify)       │        │      (NestJS)       │
   │  verifies JWTs      │        │  decodes  only  ⚠   │
   └──────┬───────┬──────┘        └──────┬───────┬──────┘
          │       │   HTTP (tracking)    │       │
          │       └─────────────────────►│       │
          │                              │       │
          │       RabbitMQ core.events   │       │
          │       ──────────────────────►│       │
          │                              │       │
          └────────────┬─────────────────┘       └──────────┐
                       ▼                                    ▼
        ┌──────────────────────────────┐        ┌──────────────────────┐
        │  PostgreSQL  (shared!)       │        │ Redis · RabbitMQ ·   │
        │  auth owns 7 RBAC tables     │        │ S3 · BullMQ queues   │
        │  core owns the other 61      │        └──────────────────────┘
        └──────────────────────────────┘
```

**All four clients point at a single base URL; the path prefix selects the service.** There is no
per-service hostname and no service discovery on the client side:

- `elearning-authentication-service/src/index.ts:48-49` registers `api/v1/auth` and `api/v1/auth/rbac`
- `elearning-core-service/src/main.ts:72` calls `app.setGlobalPrefix('api/v1/coreservice')`

So `GET https://staging.api.weupbook.com/api/v1/coreservice/blogs/list` reaches core-service and
`POST .../api/v1/auth/login` reaches auth-service. Client config holds only the host plus `/api/v1`.

### Headers every client sends

Set automatically by each client's HTTP layer. **Bypassing that layer with a raw `fetch`/`http`
call silently returns the wrong market's data** — this is a real and recurring bug.

| Header | Purpose | Set by |
|---|---|---|
| `Authorization: Bearer <jwt>` | Identity + permissions | all clients |
| `w-country` | Market selection (pricing, catalog, content) | web, mobile |
| `accept-language` / `x-lang` / `x-locale` | Response message translation | web, mobile |

Note the market/locale mismatch: the `Market` enum uses `vn`, i18n uses `vi`. core-service bridges
them with `fallbacks: { vn: 'vi' }` in its i18n module.

---

## 3. Security and the trust boundary

**Read this before deploying anything.**

`core-service` **does not verify JWT signatures.** Its
[`jwt-decode.middleware.ts`](elearning-core-service/src/presentation/middlewares/jwt-decode.middleware.ts)
base64url-decodes the token payload and assigns it straight to `req.user`:

```ts
const token = authHeader.replace(/^Bearer\s+/i, '');
const parts = token.split('.');
if (parts.length !== 3) return next();
const decoded = Buffer.from(parts[1], 'base64url').toString('utf-8');
const user = JSON.parse(decoded) as IJwtPayload;
req['user'] = user;                       // ← signature never checked
```

There is no `jwt.verify` anywhere in core's `src/`, no `jsonwebtoken` dependency, no public key and
no JWKS endpoint. The `jwt: {...}` block in `infrastructure/config/configuration.ts` reads
`JWT_SECRET`/`JWT_REFRESH_SECRET` but **nothing injects it** — it is dead config that looks live.

Consequences, stated plainly:

1. **core-service must never be reachable from the internet.** Anyone who can reach it directly can
   send an unsigned token containing `{"userId":"…","permissions":["*:*"]}` and get full admin.
2. The gateway is the entire security boundary. The JWT's `iss` claim is set from `KONG_JWT_ISS`
   (`auth-service/src/utils/generateToken.util.ts`), which implies a **Kong** gateway in front —
   but Kong's configuration lives in neither repo, so you cannot verify the boundary from here.
3. A malformed token yields an **anonymous** request, not a `401`. The `catch` block logs and calls
   `next()`, which is deliberate "optional auth" behaviour.

`auth-service` *does* verify properly — `jwt.verify` with HS256 symmetric secrets in
`src/middlewares/auth.middleware.ts`.

### Authorization in core-service is deny-by-default

`PermissionsGuard` is registered as a global `APP_GUARD` in `app.module.ts`. A handler carrying
**neither** `@Public()` **nor** `@Permissions(...)` throws `ForbiddenException` before your method
body runs — a brand-new endpoint returns `403` and people debug the controller for an hour.

Two more things that surprise people:

- `@Permissions('a:x', 'b:y')` is **OR**, not AND.
- Permissions are baked into the access token at login. Granting a permission has **no effect until
  the user re-logs in** or the 30-minute token expires.

Tokens: access 30 min, refresh 7 days, password-reset 5 min. The only mid-token-lifetime revocation
is the Redis key `blacklist:user:<userId>`, which both services check.

---

## 4. Shared API contract, in brief

Full detail in [`docs/02-api-contract.md`](docs/02-api-contract.md). The parts you cannot avoid
knowing:

**Success envelope** — `{ success, data, message, code }`, applied globally by
`ResponseTransformInterceptor`. It lifts a `message` key out of your return value and treats the
remainder as `data`, which means the two controller styles in the codebase produce **different
shapes**:

```ts
return { message, data: result };   // → { success, data: { data: result }, message, code }  ← nested!
return { message, ...result };      // → { success, data: result,          message, code }  ← flat
```

Both are live in the tree. Check the actual payload before writing client types.

**Two error shapes** ship from the same API, depending on whether the throw was an `HttpException`:

```jsonc
// I18nExceptionFilter — HttpException
{ "success": false, "data": null, "code": 404, "message": "Resource not found" }

// AllExceptionsFilter — everything else
{ "success": false, "statusCode": 500, "timestamp": "…", "path": "/api/v1/…",
  "message": "…", "details": {} }
```

**Validation is strict.** The global `ValidationPipe` runs with `forbidNonWhitelisted: true`, so
**one unexpected key in a request body is a hard `400`**. This is the single most common source of
client↔server friction on this project.

**Pagination is not standardised.** There is no shared helper — each of the ~66 list results
recomputes its own. Request side is usually `page`+`limit` but blogs use `offset`+`limit`. The
item-array key varies: `items`, `departments`, `job_descriptions`. Empty lists report
`totalPages: 0`.

---

## 5. How the two services integrate

Four channels, no shared code. `matchesPermission` is copy-pasted verbatim into both services;
the Drizzle `timestamps` spread is re-declared in all 75 schema files.

### 5.1 A shared PostgreSQL database — and a rule you must not break

Both services read the same `POSTGRES_URL` and both declare the same seven RBAC tables (`users`,
`authentications`, `roles`, `permissions`, `role_permissions`, `user_roles`,
`password_reset_tokens`). Neither sets `migrationsTable`, so **both write to the same default
`drizzle.__drizzle_migrations` ledger** in two unrelated numbering sequences (auth `0000–0010`,
core `0000–0165`).

The definitions have already diverged. core's `users` has `status`, `deleted_at` and `deleted_by`;
auth's does not.

> ### ⚠️ Table ownership
> **auth-service owns the 7 RBAC tables. core-service owns the other 61.**
>
> Run `npm run db:generate` only in the service that owns the table you changed. Running it in
> auth right now diffs auth's schema against the live database and emits
> `ALTER TABLE users DROP COLUMN status, DROP COLUMN deleted_at, DROP COLUMN deleted_by` —
> silently destroying core's user-soft-delete feature.
>
> Always read the generated SQL before applying it. **Never run `db:migrate` against production
> without a human** (core's own `CLAUDE.md` states this as a hard rule). No CI pipeline runs
> migrations — applying them is a manual, out-of-band operation.

### 5.2 RabbitMQ — auth publishes, core consumes

Exchange `core.events` (topic, durable). auth is publish-only and **fully fire-and-forget**: every
error is a `console.warn`, never thrown, and if `RABBITMQ_URL` is unset all publishes silently
no-op.

| Routing key | Payload |
|---|---|
| `core.audit.event.created` | audit log entry |
| `core.user_activity.tracked` | login / logout activity |

core's contract lives in `src/infrastructure/messaging/rabbitmq/rabbitmq.constants.ts`. Its
consumers are started by `MessagingBootstrapService`, which **swallows startup failures** — the app
boots fine with RabbitMQ down, and every event-driven feature becomes a no-op with one log line.

### 5.3 Direct HTTP — auth → core, one way only

`auth-service/src/services/tracking.service.ts` posts to
`/api/v1/coreservice/tracking/activity` with a 3 s timeout, forwarding the user's own token. Not
awaited; errors are logged. No-ops when `CORE_SERVICE_BASE_URL` is unset.

**core-service never calls auth-service.** Grepping core for `auth-service`, `AUTH_SERVICE` or port
`8000` returns nothing. The dependency is strictly one-directional.

### 5.4 Redis — a shared key convention with mismatched config

Both services read and write `blacklist:user:<userId>`. But auth reads `UPSTASH_REDIS_URL` while
core reads `REDIS_URL`/`REDIS_HOST`/`REDIS_PORT` — **different variable names for what must be one
instance.** If they diverge, cross-service blacklisting stops working silently, and auth's
in-memory `Map` fallback hides the failure completely in dev.

### 5.5 Not in this workspace

`PAYMENT_SERVICE_URL` (default `http://localhost:3436/api/v1/payment-service`) publishes to a
`payment.events` exchange that core consumes. The mobile app's Studio AI features hit a separate
Notebook host (`NOTEBOOK_API_BASE_URL`) using the same auth-service tokens. External integrations
include LadiSales, Shopify, Payoo, PayPal, Pancake POS, Base.vn, Telegram, Mailjet, Google Sheets,
GeoNames and Open Exchange Rates.

---

## 6. Quick start

Full instructions, including the env vars that actually work, are in
[`docs/09-local-setup.md`](docs/09-local-setup.md). The short version:

```bash
# Backend — needs PostgreSQL, and ideally Redis + RabbitMQ
cd elearning-core-service && npm ci && cp .env.example .env && npm run start:dev   # :3435
cd elearning-authentication-service && npm ci && cp .env.example .env && npm run dev   # :8000

# Clients
cd elearning-cms && npm ci && npm run dev                          # :4000  (env lives in env/.env)
cd elearning-fe && npm ci && npm run dev:web                       # :3000  (install from ROOT only)
cd elearning-mobile-app && make setup && make run ENV=dev
```

> ### ⚠️ `cp .env.example .env` does not give you a working service
>
> **auth-service crashes immediately.** `src/config/vars.config.ts` throws on `API_URL`,
> `MAILJET_API_KEY`, `MAILJET_API_SECRET`, `MAILJET_FROM_EMAIL` and `MAILJET_FROM_NAME` — **none of
> which are in `.env.example`.** The first error you see is `Error: API_URL is not defined`, thrown
> at import time before Fastify starts.
>
> **core-service boots but silently degrades.** Nothing validates its env. `REDIS_URL` and
> `RABBITMQ_URL` are absent from `.env.example`, so BullMQ falls back to `localhost:6379` and
> RabbitMQ is disabled with one log line. Every queue-backed feature (asset processing, order
> post-processing, enrollment, POS sync) and every consumer becomes a no-op. Treat
> `src/infrastructure/config/configuration.ts` as the real env contract; `.env.example` is a subset.

### Ports contradict themselves

| Service | Code default | Dockerfile | compose / other |
|---|---|---|---|
| core-service | **3435** | 3435 | `docker-compose.yml` publishes **4001:4001** |
| auth-service | **8000** | **5001** | — |
| cms | 4000 (`env/.env`) | nginx :80 | root `.env` says 8002 (Makefile only) |
| web | 3000 | **8001** | — |

`docker-compose up` on core-service gives you an **unreachable service**: compose supplies no
`API_PORT`, so the container listens on 3435 while the mapping targets 4001.

### Useful URLs when running locally

- core-service Swagger: `http://localhost:3435/api/v1/coreservice/docs`
- auth-service Swagger: `http://localhost:8000/api/v1/auth/docs`
- BullMQ dashboard: `http://localhost:3435/admin/queues` — **unauthenticated, and outside the
  global prefix.** So is `/docs`; `PermissionsGuard` allows any path *containing* `/docs`.

---

## 7. Client-side parallels

The three clients are architecturally parallel and deliberately share no code. Someone moving
between them reaches for the wrong name constantly, so:

| Concern | `elearning-cms` | `elearning-fe` (`apps/web`) | `elearning-mobile-app` |
|---|---|---|---|
| HTTP client | `src/core/baseClient.ts` (axios) | `src/core/utils/base-http.ts` (fetch) | `lib/core/network/api_client.dart` (dio) |
| Response type | `IResponse` — `data` is `any` | `ResponseType<T>` — generic | `Result<T>` = `Ok` \| `Err` |
| Service file | `src/services/xxxService.ts` | `src/services/xxx.api.ts` | `..._remote_data_source.dart` |
| Feature dir | `src/module/<f>/` (singular) | `src/modules/<f>/` (plural) | `lib/features/<f>/` |
| Route constants | `ROUTE_NAME` | `ROUTES` (`@elearning/constants`) | `RouteNames` |
| Server state | `useState` + manual `fetchX()` | TanStack Query + SSR hydration | Riverpod `AsyncNotifier` |
| Token storage | `localStorage` (`w_access_token`) | cookies (`refresh_token` httpOnly) | `flutter_secure_storage` |
| i18n | **none** — hardcoded Vietnamese | `next-intl`, 9 JSON files | `gen-l10n`, 9 `.arb` files |
| Error handling | resolves — check `error` | resolves — check `success` | `Result` — `fold` / `when` |
| Error toast | **automatic** in interceptor | **automatic** in interceptor | manual `AppSnackBar` |
| Permissions | RBAC from JWT, menu gated | **none** — login is binary | **none** |

Two traps this table is trying to prevent:

1. **HTTP errors never throw on cms or web.** Both clients resolve to an object carrying a failure
   flag. Wrapping a call in `try/catch` and expecting the catch to fire is wrong; check `error`
   (cms) or `success` (web).
2. **The interceptor already fired an error toast.** Adding your own produces a double toast. Pass
   `offNotify: true` to suppress the automatic one. You only ever write the *success* toast.

---

## 8. Conventions

**Branches.** Each repo is independent, so a cross-repo feature means one branch per repo, ideally
sharing a name. core-service uses `develop` as its integration branch and **merges rather than
rebases**. Current state of the workspace: auth and core on `main`, cms on `feat/thanhf`, fe on
`feat/thanhf-main`, mobile on `feat/thanhf-init`.

**Commits.** Conventional commits (`feat:`, `fix:`, `refactor:`, `docs:`).

**"Tested" means something different in every repo.** Do not promise a test in a PR description
without checking which repo you are in:

| Repo | Reality | Your verification command |
|---|---|---|
| core-service | 137 Jest specs, **excluded from lint and not run by CI** | `npm run lint && npm test` |
| auth-service | 2 Vitest files, no lint script at all | `npm test` |
| cms | **zero tests, no test infrastructure** | `npm run type-check && npm run lint` |
| fe | **zero tests**, no `test` task in `turbo.json` | `npm run type-check && npm run lint && npm run lint:css` |
| mobile | Full suite + per-file coverage floor, **enforced in CI** | `make check` |

Only the mobile app has a CI gate that can fail on code quality. core-service's CI runs neither
tests nor lint, and deploys via PM2 with `rsync --delete`.

**Node/Flutter versions disagree between local and CI.** cms `.nvmrc` says v24.12.0 but Docker and
CI use Node 20. fe `.nvmrc` says v24.13.0, Docker uses 20. core-service has no `engines` field —
its Dockerfile uses Node 22 while its CI workflow uses Node 20. Mobile pins Flutter 3.41.2 via
`.fvmrc`, and CI matches.

**Environment variables are build-time on the web clients.** Both cms (Vite) and fe (Next.js
`NEXT_PUBLIC_*`) inline env values at build time. Changing one requires a rebuild, and fe ships
**two images per release** (`.com` and `.vn`) because the domains differ.

---

## 9. Documentation index

### Start here

| Document | What it gives you |
|---|---|
| [`docs/README.md`](docs/README.md) | **The end-to-end feature workflow** — ordered phases and a master checklist |
| [`docs/01-architecture.md`](docs/01-architecture.md) | Per-project architecture and the core-service request lifecycle |
| [`docs/02-api-contract.md`](docs/02-api-contract.md) | The contract all clients depend on — envelope, errors, pagination, auth |
| [`docs/03-guide-core-service.md`](docs/03-guide-core-service.md) | Add a feature to core-service (worked example: `blog-category`) |
| [`docs/04-guide-auth-service.md`](docs/04-guide-auth-service.md) | Change auth-service, and when not to |
| [`docs/05-guide-cms.md`](docs/05-guide-cms.md) | Add an admin CRUD screen (worked example: blog posts) |
| [`docs/06-guide-web.md`](docs/06-guide-web.md) | Add a storefront page (worked example: recruit) |
| [`docs/07-guide-mobile.md`](docs/07-guide-mobile.md) | Add a Flutter feature (worked example: blog) |
| [`docs/08-walkthrough-new-feature.md`](docs/08-walkthrough-new-feature.md) | One feature traced through all five repos |
| [`docs/09-local-setup.md`](docs/09-local-setup.md) | Getting everything running, with the env vars that actually work |

### Per-repo docs worth trusting

| Document | Why |
|---|---|
| `elearning-core-service/.claude/skills/enforce-hexagonal-architecture/SKILL.md` | **The authoritative layering ruleset.** Best single source for how to build a core feature. |
| `elearning-core-service/docs/BACKEND_DEVELOPER_HANDOVER.md` | 908 lines, accurate. §3 is a 12-step feature guide. |
| `elearning-core-service/README.md` | Correct on hexagonal + use-case architecture (counts are stale). |
| `elearning-core-service/wiki/` | 28 feature pages. Start at `wiki/index.md`. See also `wiki/debt/known-issues.md`. |
| `elearning-mobile-app/ARCHITECTURE.md` + `guideline-core-convention/00..08-*.md` | The mobile rulebook — naming, network, state, UI, l10n, testing. |
| `elearning-mobile-app/lib/features/_feature_template/` | A copy-me feature skeleton. |
| `elearning-fe/README.md` + `HANDOFF.md` | Conventions and honest status, including "no test infra". |
| `elearning-cms/.cursor/rules/elearning-cms-frontend.mdc` + `UIRULE.md` | Architecture rules and UI guardrails. |

### ⚠️ Superseded — these will actively mislead you

| Document | The problem |
|---|---|
| `elearning-core-service/CLAUDE.md` and `AGENT.md` (byte-identical) | Document a **CQRS architecture that was removed**: `CommandBus`, `QueryBus`, a `src/core/` directory. None exist. The `@core/*` tsconfig alias points at nothing. Its "Adding New Features" steps name files that no longer exist. |
| `elearning-authentication-service/AGENTS.md` | Claims strict ESM, `src/routes/`, `src/lib/`, a placeholder `npm test`. Reality: CommonJS, `src/controllers/`, `src/services/`, Vitest. |
| `elearning-mobile-app/ARCHITECTURE.md` §2.1 / §14, `README.md` §3 | Wrong Android `applicationId`, lists 9 features where there are 19, and says screens "still use mock data" — largely untrue now. |
| Any use-case / table count in an existing doc | core's README says 153 use cases / 46 tables; its wiki says 222 / 60. Actual: **275 / 68**. |

**These files are not modified by this documentation set** — they live in five separate repos on
active feature branches. Fixing them is its own PR in each repo. Until then, prefer the source.
