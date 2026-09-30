# Running everything locally

The env vars listed here are the ones the code **actually requires**, read out of
`vars.config.ts` and `configuration.ts` — not copied from `.env.example`, which does not boot either
service. See §2 and §3.

---

## 1. Prerequisites

| Tool | Version | Notes |
|---|---|---|
| Node | **20 or 22** | See the mismatch table below before picking |
| npm | ≥ 10 | All four Node projects use npm |
| PostgreSQL | 15+ | **One database shared by both services** |
| Redis | any recent | Optional but strongly recommended — see §3 |
| RabbitMQ | any recent | Optional — without it every event feature silently no-ops |
| Flutter | **3.41.2** | Pinned in `.fvmrc`. Use fvm. |
| Xcode / Android Studio | current | For the mobile app |

### Node version disagreements

| Project | `.nvmrc` / `engines` | Docker | CI |
|---|---|---|---|
| core-service | *(none)* | **22** | **20** |
| auth-service | `>=22.0.0` | 22 | *(no CI)* |
| cms | `.nvmrc` **v24.12.0** | **20** | **20** |
| fe | `.nvmrc` **v24.13.0**, `engines >=20` | **20** | `.nvmrc` |

**Node 22 is the safest single choice** for the two backends. If you hit a build difference in a
client, match its Docker version (20) rather than its `.nvmrc`.

### Infrastructure with Docker

Neither service ships a usable compose file for dependencies (core's has a commented-out Postgres
block), so run them directly:

```bash
docker run -d --name weup-pg    -p 5432:5432 \
  -e POSTGRES_PASSWORD=postgres -e POSTGRES_DB=elearning postgres:15-alpine
docker run -d --name weup-redis -p 6379:6379 redis:7-alpine
docker run -d --name weup-rmq   -p 5672:5672 -p 15672:15672 rabbitmq:3-management-alpine
```

---

## 2. auth-service

```bash
cd elearning-authentication-service
npm ci
```

> ### ⚠️ `cp .env.example .env` then `npm run dev` crashes immediately
> `src/config/vars.config.ts` validates at import time and throws on five variables that
> `.env.example` does not contain. The first error is `Error: API_URL is not defined`, raised before
> Fastify starts because `swagger.config.ts` imports `vars.config` at module scope.

Write `.env` by hand. **All eleven of these are hard requirements** — the service will not boot
without them:

```bash
FE_URL=http://localhost:3000
GOOGLE_OAUTH2_REDIRECT_PATH=/api/v1/auth/login/google
GOOGLE_OAUTH2_CALLBACK_PATH=/api/v1/auth/login/google/callback
GOOGLE_CLIENT_ID=placeholder
GOOGLE_CLIENT_SECRET=placeholder
GOOGLE_IOS_CLIENT_ID=placeholder
API_URL=http://localhost:8000          # ← NOT in .env.example
MAILJET_API_KEY=placeholder            # ← NOT in .env.example
MAILJET_API_SECRET=placeholder         # ← NOT in .env.example
MAILJET_FROM_EMAIL=dev@example.com     # ← NOT in .env.example
MAILJET_FROM_NAME=WEUP Dev             # ← NOT in .env.example
```

Placeholders are fine for the Google and Mailjet ones unless you are testing OAuth or email — the
validator only checks presence.

Then the rest, which have defaults or fail quietly:

```bash
POSTGRES_URL=postgres://postgres:postgres@localhost:5432/elearning
JWT_ACCESS_TOKEN_SECRET_KEY=dev-access-secret
JWT_REFRESH_TOKEN_SECRET_KEY=dev-refresh-secret
SESSION_SECRET=dev-session-secret
API_PORT=8000
CORE_SERVICE_BASE_URL=http://localhost:3435   # optional — tracking no-ops without it
RABBITMQ_URL=amqp://guest:guest@localhost:5672
UPSTASH_REDIS_URL=redis://localhost:6379      # falls back to an in-memory Map
KONG_JWT_ISS=weup-local                       # in no example file; becomes the JWT `iss`
COOKIE_SECRET=dev-cookie-secret               # defaults to the literal 'supersecret'
NODE_ENV=development
```

```bash
npm run dev      # → http://localhost:8000   docs at /api/v1/auth/docs
```

## 3. core-service

```bash
cd elearning-core-service
npm ci
cp .env.example .env
```

`.env.example` works here — but understand what it leaves out.

> ### ⚠️ core-service boots successfully while silently degraded
> `ConfigModule.forRoot` has **no `validationSchema`**, so nothing validates its env.
> `REDIS_URL` and `RABBITMQ_URL` are **absent from `.env.example`**, which means:
> - BullMQ falls back to `localhost:6379`
> - RabbitMQ is **disabled** with one log line: `"RabbitMQ is disabled because RABBITMQ_URL is not set"`
>
> Every queue-backed feature (asset processing, order post-processing, manual enrollment, POS sync)
> and every consumer (audit log, user activity, payment events) becomes a **no-op with no error.**
> `MessagingBootstrapService` swallows startup failures by design.
>
> Treat `src/infrastructure/config/configuration.ts` as the real env contract.

Add at minimum:

```bash
POSTGRES_URL=postgres://postgres:postgres@localhost:5432/elearning
API_PORT=3435
NODE_ENV=development
REDIS_URL=redis://localhost:6379                   # ← not in .env.example
RABBITMQ_URL=amqp://guest:guest@localhost:5672     # ← not in .env.example
RABBITMQ_QUEUE_PREFIX=local                        # ← not in .env.example; defaults to 'staging'
WEB_URL=http://localhost:3000
```

Also missing from `.env.example` but read by `configuration.ts`: `MAILJET_SUPPORT_EMAIL`,
`POS_API_KEY_ENCRYPTION_SECRET`, `MASTER_SECRET_PASS_KEY`, `TELEGRAM_BOT_TOKEN`, `TELEGRAM_CHAT_ID`,
`DEFAULT_LOCALE`, `FALLBACK_LOCALE`, `SUPPORTED_LOCALES`, and the `S3_*` spellings.

**Dead config that looks live** — do not waste time filling these in: `MONGO_URL` and `DB_NAME`
(there is no Mongo anywhere), `RECAPTCHA_*`, `POSTMARK_*`, `CKO_TIMEOUT`, and the whole
`JWT_SECRET` / `JWT_REFRESH_SECRET` / `JWT_*_EXPIRATION` block — core-service never verifies a token,
so nothing injects it.

Note S3 keys are read as `process.env.S3_X || process.env.AWS_S3_X`, so **both spellings work**, but
only the `AWS_*` spelling is in `.env.example`.

Cosmetic but confusing: every line after the first in core's `.env.example` is **indented two
spaces**. dotenv trims it so it parses, but `grep '^VAR='` finds nothing.

```bash
npm run start:dev    # → http://localhost:3435   docs at /api/v1/coreservice/docs
```

### Migrations and seeds

Run these once, **in this order**, against an empty database:

```bash
# 1. RBAC tables — from the service that OWNS them
cd elearning-authentication-service
npm run db:migrate

# 2. everything else
cd ../elearning-core-service
npm run db:migrate

# 3. roles, permissions and login accounts
npm run seed:roles
npm run seed:permissions
```

> ### ⚠️ Read before running any migration command
> - **Only run `db:generate` in the service that owns the table.** auth owns the 7 RBAC tables; core
>   owns the other 61. Running it in the wrong repo emits `DROP COLUMN` against the other service's
>   columns. Full explanation:
>   [`../README.md`](../README.md#51-a-shared-postgresql-database--and-a-rule-you-must-not-break).
> - **Always read the generated SQL** before `db:migrate`.
> - **Never `db:migrate` against production without a human.** core's `CLAUDE.md` states this as a
>   hard rule, and no CI pipeline runs migrations.
> - `scripts/restore-drizzle-migrations.ts` (`npm run db:restore-migration-history`) exists to repair
>   a corrupted ledger — which tells you this has already gone wrong at least once.
> - Migration numbering has already collided: there are **two `0164_*.sql` files**, disambiguated only
>   by `meta/_journal.json`, so the file prefix and the journal index are off by one from `0164`
>   onward. `git pull` before generating, and never rename a migration.

### Seed accounts

`npm run seed:roles` creates four roles — `admin`, `content_creator`, `operator`,
`operator_manager` — grants each the `*:*` permission, and creates four users:

| Email | Role |
|---|---|
| `admin@weupbook.com` | admin |
| `contentcreator@weupbook.com` | content_creator |
| `operator@weupbook.com` | operator |
| `operatormanager@weupbook.com` | operator_manager |

**All four share the hardcoded password `Admin@123` and have `*:*`.** Fine locally; change them
before any shared environment — `npm run change:password` exists for this.

### Four npm scripts in core-service are dead

They are defined in `package.json` but their target files do not exist. Running them fails with a
module-not-found error:

| Script | Missing file | Use instead |
|---|---|---|
| `npm run seed` | `scripts/seed.ts` | `seed:roles` then `seed:permissions` |
| `npm run backfill` | `scripts/backfill-product-fields.ts` | one of the specific `backfill:*` scripts |
| `npm run test:i18n` | `test-i18n.sh` (it is in the **auth** repo) | — |
| `npm run frontend` | `frontend/server.js` | — |

---

## 4. elearning-cms

```bash
cd elearning-cms
npm ci
```

> ### ⚠️ Env lives in `env/`, not the repo root
> `vite.config.ts` sets `envDir: './env'`, so `npm run dev` reads **`env/.env`** — git-ignored, so
> you must create it. The root `elearning-cms/.env` is read **only by the `Makefile`** for Docker
> build args, and specifies a different port (8002 vs 4000).

```bash
# env/.env  — the complete variable list
VITE_APP_PORT=4000
VITE_APP_URL=http://localhost:4000
VITE_API_URL=https://staging.api.weupbook.com     # or http://localhost:3435 for a local backend
VITE_APP_WEB_DOMAIN=https://staging.weupbook.com
```

```bash
npm run dev      # → http://localhost:4000
```

Log in with a seed account. Note `pre-push` runs `type-check` **and** a full `build`, so pushing is
slow.

## 5. elearning-fe

```bash
cd elearning-fe
npm ci                      # ⚠ from the REPO ROOT — never inside apps/web
cp apps/web/.env.example apps/web/.env
npm run dev:web             # → http://localhost:3000
```

`src/config/env.config.ts` validates with zod at import time and **throws on any missing
variable**, so copy `.env.example` first. The eleven required vars:

```
NEXT_PUBLIC_SERVER_API_URL      NEXT_PUBLIC_APP_URL
NEXT_PUBLIC_NOTEBOOK_API_URL    NEXT_PUBLIC_ASSISTCHAT_API_URL
NEXT_PUBLIC_GA_MEASUREMENT_ID   NEXT_PUBLIC_BLOCK_DEVTOOL_MODE
NEXT_PUBLIC_IP_DETECT_ENDPOINT  NEXT_PUBLIC_IP_DETECT_TOKEN
NEXT_PUBLIC_COMPANY_PROFILE_PDF_KEY
NEXT_PUBLIC_COPILOTKIT_PUBLIC_API_KEY
NEXT_PUBLIC_EXPERT_CHATBOT_URL
```

Two notes:

- **Install from the root.** The `dev` script copies
  `../../node_modules/pdfjs-dist/legacy/build/pdf.worker.min.mjs` into `public/`, which only resolves
  with a hoisted root install. Without it the PDF reader fails with "PDF information is not
  available". `postinstall` normally handles the copy, but note the script uses a single `&`, so on a
  cold clone it can race.
- **`apps/workspace` needs a subdomain.** Plain `localhost:3001` resolves no tenant and 404s. Use
  `demo.localhost:3001`, `demo.lvh.me:3001`, or set `NEXT_PUBLIC_DEV_PARTNER_SLUG` +
  `WORKSPACE_DEV_TENANTS`.

## 6. elearning-mobile-app

```bash
cd elearning-mobile-app
make setup                  # flutter pub get + codegen — REQUIRED, a fresh clone won't compile
make run ENV=dev
```

> **Never run `flutter run` bare.** Without `--dart-define-from-file` the app falls back to
> `http://localhost:4000/api/v1` and silently talks to nothing. Always `make run ENV=…`.
>
> **Re-run `make gen` after every pull** and after any change to an ARB file, a Freezed model or an
> annotation. `*.freezed.dart`, `*.g.dart` and `lib/l10n/app_localizations*.dart` are all gitignored.

To point at a local backend:

```bash
cp config/dev.local.example.json config/dev.local.json
```

| Target | `API_BASE_URL` |
|---|---|
| Android emulator | `http://10.0.2.2:4000/api/v1` — **emulators cannot reach `localhost`** |
| iOS simulator | `http://localhost:4000/api/v1` |
| Physical device | `http://192.168.1.x:4000/api/v1` |

`config/dev.local.json` is layered **after** `config/dev.json`, so it only needs the keys it
overrides.

Platform first-run: **iOS** needs `pod install` in `ios/`, and debug/JIT is blocked on some physical
devices — use `make run-profile`. **Android** release builds are currently signed with the debug key;
you need `android/key.properties` and a real keystore before shipping.

`ENV=prod` is blocked on purpose — `config/prod.json` holds `api.replace-me.invalid`, and both
`make check-env` and `Env.validate()` refuse it.

---

## 7. Ports

| Service | Code default | Dockerfile | Elsewhere |
|---|---|---|---|
| core-service | **3435** | 3435 | `docker-compose.yml` publishes **4001:4001** |
| auth-service | **8000** | **5001** | — |
| cms | **4000** (`env/.env`) | nginx :80 | root `.env` says 8002 |
| fe web | **3000** | **8001** | — |
| fe workspace | **3001** | — | — |
| PostgreSQL | 5432 | | |
| Redis | 6379 | | |
| RabbitMQ | 5672 (15672 UI) | | |

> **`docker-compose up` on core-service gives an unreachable service.** Compose supplies no
> `API_PORT`, so the container listens on 3435 while the mapping targets 4001. Either add
> `API_PORT=4001` to your `.env` or change the mapping to `3435:3435`.

### URLs

| | |
|---|---|
| core-service Swagger | `http://localhost:3435/api/v1/coreservice/docs` |
| auth-service Swagger | `http://localhost:8000/api/v1/auth/docs` |
| BullMQ dashboard | `http://localhost:3435/admin/queues` — **unauthenticated** |
| RabbitMQ management | `http://localhost:15672` (guest/guest) |

Both `/docs` and `/admin/queues` sit outside the global prefix and are unguarded. `PermissionsGuard`
allows any path *containing* `/docs` — a substring match, not a prefix match.

---

## 8. Verification commands

"Tested" means something different in each repo. Run the right one:

| Repo | Command | What actually runs |
|---|---|---|
| core-service | `npm run lint && npm test` | ESLint + 137 Jest specs. **Specs are excluded from lint and CI runs neither.** |
| auth-service | `npm test` | Vitest — **only 2 test files exist.** No lint script. |
| cms | `npm run type-check && npm run lint` | **No tests at all.** |
| fe | `npm run type-check && npm run lint && npm run lint:css` | **No tests, no `test` task in `turbo.json`.** |
| mobile | `make check` | gen + format + analyze + full test suite + coverage gate. **Enforced in CI.** |

Only the mobile app has a CI gate that can fail on code quality.

---

## 9. Troubleshooting

| Symptom | Cause |
|---|---|
| `Error: API_URL is not defined` | auth-service — the five vars missing from `.env.example`. §2. |
| Queue or event feature does nothing, no error | `RABBITMQ_URL` unset. §3. |
| `docker-compose up` service unreachable | The 3435 vs 4001 mismatch. §7. |
| Every core-service endpoint returns 403 | Missing `@Public()`/`@Permissions()`, or the permission was never seeded. |
| A permissioned endpoint 401s with a valid token | Controller missing from `app.module.ts` `forRoutes(...)`. |
| Permission granted but still 403 | Permissions are baked into the JWT — **log out and back in.** |
| `message` is the literal `common.foo.bar` | Missing i18n key in one of the 8 locale files. |
| Relational query fails, migration was fine | Table missing from `schemas/index.ts`. |
| `db:generate` wants to drop columns | You ran it in the wrong service. **Do not apply it.** §3. |
| cms: `npm run dev` ignores your env | You edited the root `.env` instead of `env/.env`. §4. |
| fe: throws on boot about env vars | `apps/web/.env` not created. §5. |
| fe: PDF reader says "information is not available" | `npm ci` was run inside `apps/web`. §5. |
| fe: can't find `middleware.ts` | It is `src/proxy.ts` (Next.js 16 rename). |
| mobile: wall of missing-file errors | Codegen not run. `make setup`, then `make gen`. |
| mobile: app reaches no backend | `flutter run` used bare. Use `make run ENV=dev`. |
| mobile: emulator can't reach the API | Use `10.0.2.2`, not `localhost`. §6. |
| mobile: `check_l10n` fails | A key is missing from one of the 9 `.arb` files. |

---

## Appendix — re-deriving the counts

Any count quoted in these documents can be regenerated. Run from the workspace root:

```bash
C=elearning-core-service
find $C/src -name '*.use-case.ts' ! -name '*.spec.ts' | wc -l    # use cases      → 275
find $C/src -name '*.controller.ts' ! -name '*.spec.ts' | wc -l  # controllers    → 50
find $C/src/infrastructure/databases/schemas -name '*.schema.ts' | wc -l   # tables → 68
ls $C/src/infrastructure/databases/migrations/*.sql | wc -l      # migrations     → 172
find $C/src -name '*.spec.ts' | wc -l                            # specs          → 137
ls -d $C/src/domain/*/ | wc -l                                   # bounded ctx    → 27
ls $C/src/infrastructure/i18n/locales                            # locales        → 8

ls elearning-fe/apps/web/messages/*.json | wc -l                 # web locales    → 9
ls elearning-mobile-app/lib/l10n/*.arb | wc -l                   # mobile locales → 9
ls -d elearning-mobile-app/lib/features/*/ | wc -l               # features       → 19
ls -d elearning-cms/src/module/*/ | wc -l                        # cms modules    → 16
ls -d elearning-fe/apps/web/src/modules/*/ | wc -l               # web modules    → 26
```

Values on the right were correct on **2026-09-30**. The existing per-repo docs quote stale figures
(core's README says 153 use cases / 46 tables; its wiki says 222 / 60), which is why these commands
are here rather than another hardcoded table.
