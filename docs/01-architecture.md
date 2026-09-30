# Architecture

One section per project. Each says what the layers are, where every kind of file lives, and which
rules the codebase actually enforces.

---

## 1. core-service — hexagonal / ports & adapters

`elearning-core-service` is the largest and most disciplined codebase here: **275 use cases,
50 controllers, 68 tables, 137 specs**. It is organised in four layers with dependencies pointing
strictly inward.

```
presentation  ──►  application  ──►  domain  ◄──  infrastructure
   (HTTP)           (use cases)      (pure)        (DB, queues, S3)
```

The authoritative statement of these rules is
`elearning-core-service/.claude/skills/enforce-hexagonal-architecture/SKILL.md`. Read it before
your first feature.

```
src/
├── main.ts                 # bootstrap: CORS, ValidationPipe, interceptor, filters, prefix, Swagger
├── app.module.ts           # 400+ lines: every module import + the middleware forRoutes() list
├── app.module.spec.ts      # regression test asserting middleware wiring
│
├── domain/                  # 27 bounded contexts — PURE, zero framework imports
│   └── <context>/
│       ├── entities/*.entity.ts
│       ├── value-objects/*.vo.ts
│       ├── ports/repositories/*.repository.port.ts   # interface + DI Symbol
│       ├── ports/services/*.service.port.ts
│       └── exceptions/*.exception.ts
│
├── application/             # 44 feature folders — orchestration only
│   └── <feature>/use-cases/<verb-noun>/
│       ├── <verb-noun>.command.ts     # mutations
│       ├── <verb-noun>.query.ts       # reads
│       ├── <verb-noun>.result.ts      # + static fromEntity()
│       ├── <verb-noun>.use-case.ts    # @Injectable, execute()
│       └── <verb-noun>.use-case.spec.ts
│
├── infrastructure/
│   ├── databases/{drizzle.provider.ts, schemas/*.schema.ts, migrations/*.sql}
│   ├── persistence/drizzle/repositories/*.repository.adapter.ts
│   ├── di/*.module.ts               # binds port Symbol → adapter; registers use cases
│   ├── interceptors/response-transform.interceptor.ts
│   ├── filters/{i18n-exception,all-exceptions}.filter.ts
│   ├── i18n/locales/<lang>/common.json
│   ├── messaging/rabbitmq/, queue/, storage/, mail/, logger/, config/
│
├── presentation/
│   ├── http/rest/admin/<feature>/        # back-office, routes prefixed admin/
│   ├── http/rest/web/<feature>/          # consumer-authenticated
│   ├── http/rest/integration/<feature>/  # partner API, guarded by API key
│   ├── http/rest/<feature>/              # public / shared
│   ├── guards/permissions.guard.ts       # the single global RBAC guard
│   └── middlewares/{jwt-decode,get-market}.middleware.ts
│
└── shared/{constants,decorators,helpers,utils}/
```

### Where each artifact type lives

| Artifact | Location |
|---|---|
| HTTP DTOs (validation + Swagger) | `presentation/http/rest/<feature>/dtos/*-http.dto.ts` |
| Commands / queries / results | `application/<feature>/use-cases/<uc>/*.{command,query,result}.ts` |
| Domain entities | `domain/<ctx>/entities/*.entity.ts` |
| Value objects | `domain/<ctx>/value-objects/*.vo.ts` |
| Repository **interface** + DI Symbol | `domain/<ctx>/ports/repositories/*.repository.port.ts` |
| Repository **implementation** | `infrastructure/persistence/drizzle/repositories/*.repository.adapter.ts` |
| Drizzle table schema | `infrastructure/databases/schemas/*.schema.ts` |
| DI wiring | `infrastructure/di/<feature>.module.ts` |
| Permission constants | `shared/constants/permissions.ts` |
| Decorators | `shared/decorators/{permissions,public}.decorator.ts` |
| Domain exceptions | `domain/<ctx>/exceptions/`, base in `domain/shared/exceptions/domain-exceptions.ts` |
| Per-route guards | co-located, e.g. `.../webhook/guards/shopify-hmac.guard.ts` |

### Request lifecycle

Tracing `POST /api/v1/coreservice/blog-categories`:

```
1.  Express                  strips the global prefix api/v1/coreservice
2.  JwtDecodeMiddleware      base64-decodes the JWT payload → req.user  (NO signature check)
                             + checks Redis blacklist:user:<id>
3.  GetMarketMiddleware      w-country → cf-ipcountry → DEFAULT_MARKET → req.market
       ⚠ steps 2–3 run ONLY for controllers listed in app.module.ts configure().forRoutes(...)
4.  PermissionsGuard         /docs? → allow. @Public()? → allow.
                             no @Permissions metadata? → 403. no req.user? → 401.
                             OR-match permissions against the JWT's array.
5.  ValidationPipe           class-validator on the DTO.
                             whitelist: true, forbidNonWhitelisted: true → unknown key = 400
6.  Controller               builds a Command, calls useCase.execute()
7.  Use case                 Load → Execute → Save → Return. Talks to the PORT, not the adapter.
8.  Repository adapter       Drizzle SQL. Entity.reconstitute() on read, never .create()
9.  ResponseTransform        wraps in { success, data, message, code }, translates the message
    (on error)               I18nExceptionFilter or AllExceptionsFilter — two different shapes
```

### The rules that are actually enforced

From `SKILL.md`, plus what breaks at runtime if you ignore them:

- **`domain/` imports nothing from a framework** — no `@nestjs/*`, no `drizzle-orm`, no
  `class-validator`, no Swagger, no DB schemas. (Domain exceptions importing `HttpStatus` is a
  grandfathered exception.)
- **Repository adapters use `Entity.reconstitute()`, never `Entity.create()`.** `create()` calls
  `Id.generate()` and would mint a fresh UUID on every read.
- **`mapToDomain` must return the entity directly.** A documented production bug came from a
  repository that round-tripped through a DTO, so `id` became a plain string and
  `book.id.getValue is not a function` blew up at runtime.
- **`findById` returns `null` on a miss.** It does not throw.
- **Persist value objects with `.getValue()`.**
- **`class-validator` belongs only in presentation DTOs.**
- **`forwardRef()` is banned.** The fix is a thin shared DI module — `hr-shared.module.ts`,
  `cart-repository.module.ts` and `ebook-repository.module.ts` are the existing examples.
- Multi-table writes need a transaction via `ITransactionManagerPort`.

### Two DI-token conventions coexist

1. **Legacy central** — `shared/constants/tokens.ts`, ~25 symbols.
2. **Current** — the token is declared inside the port file next to its interface:
   ```ts
   export const BLOG_CATEGORY_REPOSITORY = Symbol('BLOG_CATEGORY_REPOSITORY');
   ```

**Use convention 2 for new features.** Naming is inconsistent even within it — some tokens end
`_REPOSITORY`, some `_REPOSITORY_PORT`. Match the neighbours in your bounded context.

### Legacy you should not copy

- `presentation/modules/upload/` — the last survivor of the pre-hexagonal layout.
- `@nestjs/cqrs` is still in `package.json` but **unused for features**. Any doc mentioning
  `CommandBus`/`QueryBus` is describing a removed architecture.
- `src/presentation/http/rest/department/` is a complete CRUD but is indented with 4 spaces against
  a 2-space `.prettierrc`, has zero tests, and its controller injects a use case from another
  feature. Copy `blog-category` instead.

---

## 2. auth-service — flat layered

`elearning-authentication-service` is deliberately small: ~40 source files, no DI container, no
repository layer. Services are module-level singletons imported by name.

```
controller  ──►  service  ──►  drizzle db
```

```
src/
├── index.ts                  # Fastify bootstrap — the ONLY place routes are registered
├── config/
│   ├── vars.config.ts        # fail-fast env validation — THROWS at import time
│   ├── i18n.config.ts        # exports an already-invoked promise; index.ts must await it
│   ├── oauth2.config.ts      # @fastify/oauth2 Google config, PKCE S256
│   └── swagger.config.ts     # routePrefix /api/v1/auth/docs
├── controllers/{auth,rbac}.controller.ts        # 12 + 17 routes
├── services/                 # auth (658 lines), rbac, mail, cache, rabbitmq, tracking, audit
├── middlewares/              # auth (jwt.verify), rbac (requirePermission), error/successHandler
├── schemas/                  # TypeBox request/response schemas + createRouteSchema()
├── db/{index.ts, schemas/ (7 tables), migrations/ (11 SQL), seeds/rbac-seed.ts}
├── locales/{en,vi}/{errors,success}.json
└── utils/                    # generateToken, hashPassword, otp, error (throwError), rbac.utils
```

Differences from core-service that will trip you up:

- **Validation is TypeBox**, not class-validator.
- **Responses are explicit**, not intercepted — controllers call `successHandler(...)`, which
  always replies `200`.
- **Errors come from `throwError(translationKey, statusCode, params)`**, handled by a single
  `app.setErrorHandler`.
- **Guards are per-route `preHandler` arrays**, not a global guard. Two styles exist in the tree
  (array form in `rbac.controller.ts`, sequential-await form in `auth.controller.ts`).
- **No lint script at all** — Prettier only.
- Language is parsed from `accept-language` only. Unlike core, auth does **not** honour
  `x-lang`/`x-locale`.
- `i18n.config.ts` declares 4 namespaces but only 2 files per locale exist; `common` and `auth`
  resolve to nothing.

---

## 3. elearning-cms — feature-sliced SPA

React 19 + Vite (the `rolldown-vite` fork) + Ant Design 6 + Tailwind 4 + Zustand. **No React
Query** — data fetching is `useState` plus a manual `fetchX()` in a custom hook. **No form library**
beyond antd's own `Form`. **No i18n** — Vietnamese strings are written directly in JSX. **No tests.**

```
src/
├── core/                    # baseClient.ts (axios + interceptors + refresh queue), cmsClient.ts
├── router/{index.ts, routes.tsx}     # routes.tsx composes EVERY module's route array
├── layout/
│   ├── AuthRequiredLayout.tsx        # SideBar + RoleBasedRouteGuard + AssetUploadTracker
│   └── components/{SideBar.tsx, RoleBasedRouteGuard.tsx}
├── module/<feature>/        # 16 modules
│   ├── route.tsx            # lazy imports + RouteObject[]
│   ├── pages/  components/  modals/  hooks/  constants/  types/
├── services/                # 20 class-based singletons: <domain>Service.ts
├── store/                   # 11 zustand stores
└── shared/{components,constants,helpers,hooks,types,utils}/
```

Architecture: **feature-sliced modules over a layer-sliced service layer.** Shared components carry
a `W` prefix (`WEditor`, `WImageUpload`, `WHeaderForm`).

Authentication is `localStorage` + a react-router `loader` guard; authorization is RBAC decoded from
the JWT, gating both routes (`ROUTE_PERMISSION_MAP`) and the sidebar (`AllMenuItems`).

---

## 4. elearning-fe — Turborepo + Next.js App Router

```
elearning-fe/
├── apps/web/            @elearning/web        — the storefront (weupbook.com)
├── apps/workspace/      @elearning/workspace  — B2B partner portal, ~14 files, early scaffold
└── packages/
    ├── types/       @elearning/types      — cross-app TS types
    ├── constants/   @elearning/constants  — ROUTES, QUERY_KEYS, country/market, antd rules
    └── utils/       @elearning/utils      — cn, date, jwt, money, eventBus, …
```

Packages have **no build step** — Next compiles their TypeScript source via `transpilePackages`.

```
apps/web/src/
├── app/                     # App Router — thin wiring only, Server Components by default
│   ├── [locale]/…           # every user-facing route lives under the locale segment
│   └── api/                 # route handlers acting as a BFF: auth/{login,logout,refresh-token}
├── proxy.ts                 # ⚠ Next.js 16's renamed middleware — auth guard + GeoIP + next-intl
├── core/utils/base-http.ts  # fetch-based HTTP client (~530 lines)
├── services/                # 22 × <domain>.api.ts
├── modules/<feature>/       # 26 modules — pages, components, hooks, types, store, modals
├── shared/
│   ├── queries/             # TanStack Query hooks
│   ├── hooks/               # useApiMutation, useMessage, usePageParam, …
│   ├── store/               # zustand
│   └── {constants,types,utils}/   # mostly re-export bridges over the workspace packages
└── i18n/routing.ts          # next-intl routing + navigation wrappers
messages/{vn,en,us,th,ph,my,id,mx,br}.json   # 9 locales
```

Two things about this project catch everyone:

- **The middleware is `src/proxy.ts`, not `middleware.ts`.** Next.js 16 renamed it. Looking for
  `middleware.ts`, not finding it, and concluding there is no route guard is the classic mistake.
- **`src/shared/` is a re-export bridge, not a home for new code.** `shared/constants/index.ts` is
  literally `export * from '@elearning/constants'`. Genuinely shared code belongs in
  `packages/`; only app-local things live in `shared/`.

State is split by origin: **server state in TanStack Query** (with SSR prefetch → `dehydrate` →
`<HydrationBoundary>`), **client state in Zustand.** Auth is cookie-based and brokered through the
app's own route handlers so the refresh token can be `httpOnly`. There are **no roles or
permissions** here — login is binary.

---

## 5. elearning-mobile-app — feature-first MVVM

Flutter 3.41 + **Riverpod 3 as both state management and DI**. Deliberately *not* used: bloc,
provider, get_it, retrofit, auto_route, hive, easy_localization, melos, build flavours.

**There are no use-case files.** `usecases/` exists only as an empty placeholder in the feature
template — controllers call repositories directly, which the architecture doc sanctions.

```
lib/
├── main.dart                # bootstrap + ProviderContainer + overrides
├── app/
│   ├── bootstrap.dart       # binding, Env.validate(), error boundaries
│   ├── config/env.dart      # the ONLY place reading String.fromEnvironment
│   ├── router/{app_router.dart, route_names.dart}
│   └── shell/main_navigation_shell.dart   # bottom nav + persistent audio mini player
├── core/                    # infrastructure only — no product UI, no feature logic
│   ├── network/             # api_client, api_endpoints, auth_interceptor, market_interceptor,
│   │                        # api_failure_mapper, network_providers, single_flight, …
│   ├── result/result.dart   # sealed Result<T> = Ok<T> | Err<T>
│   ├── errors/failure.dart  # sealed Failure hierarchy
│   ├── logging/, storage/
├── shared/                  # cross-feature: theme, widgets, playback engines, providers, stores
├── features/<feature>/      # 19 features
│   ├── <feature>_providers.dart          # DI composition root
│   ├── data/{datasources,models,mappers,repositories}/
│   ├── domain/{entities,repositories}/
│   └── presentation/{controllers,screens,widgets}/
└── l10n/*.arb               # 9 locales; generated Dart output is gitignored
```

Dependency rules (`guideline-core-convention/01-architecture-rules.md`, DEP-1..8):

```
presentation  ──►  domain  ◄──  data  ──►  core
```

- `domain` imports no Flutter, Riverpod, Dio, `data` or `presentation`
- `data` never imports `presentation`
- `presentation` imports the repository **contract**, never the impl
- `core` never imports `features`, and may read only `app/config/env.dart` from `app`
- a feature never imports another feature's `data` or `presentation`
- only `app/router` imports feature pages

**No barrel files.** Every import is an explicit `package:weup_elearning/...` path —
`always_use_package_imports` makes relative imports a lint error.

**Codegen output is gitignored.** `*.freezed.dart`, `*.g.dart` and `lib/l10n/app_localizations*.dart`
are all generated. A fresh clone does not compile until `make setup`.

---

## 6. Deployment

| Project | How it ships |
|---|---|
| core-service | GitHub Actions on push to `develop` → build, tar, SCP to staging, **PM2 restart**. No test, lint or migration step. |
| auth-service | No workflow. `Makefile` builds and pushes to `registry.weupbook.com/auth-service`. |
| cms | Actions on push to `develop` → `vite build` → SCP `dist` to an nginx path. Docker image via `Makefile`. |
| fe | Actions on push to `dev` → `build:web` → repackage the standalone bundle → SCP. **Two images per release** (`.com` and `.vn`) because `NEXT_PUBLIC_*` is inlined at build time. |
| mobile | `make build-apk ENV=prod` / `build-appbundle` / `build-ios`, all obfuscated with split debug info. CI runs the full check gate on PRs to `main`. |

Migrations are applied **manually, out of band, by a human.** No pipeline runs them.
