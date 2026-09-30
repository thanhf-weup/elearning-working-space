# Building a feature end-to-end

This is the ordered procedure for shipping a feature that spans the backend and one or more
clients. It exists because the five projects are **separate git repositories** — you cannot make a
single atomic change across them, so the order of work and the handoffs between phases matter.

Read [`../README.md`](../README.md) first if you have not. It covers the system map, the trust
boundary, and the shared-database ownership rule.

---

## The shape of the work

```
  Phase 0          Phase 1           Phase 2          Phase 3
  CONTRACT   →   core-service   →   auth-service  →   CLIENTS (parallel)
                                     (only if                ├── cms
                                      needed)                ├── web
                                                             └── mobile
  ─────────────────────────────────────────────────────────────────────────
  artifact:      artifact:          artifact:         artifact:
  endpoint       deployed           deployed          released client
  spec           endpoint +         endpoint          builds
                 Swagger
```

Three rules that follow from the repos being separate:

1. **Never start a client before its endpoint is deployed to staging.** Client work against an
   un-deployed endpoint cannot be verified, and you will discover the response shape is different
   from what you assumed. The `data.data` nesting trap in
   [`02-api-contract.md`](02-api-contract.md#the-datadata-trap) catches people this way constantly.
2. **Write the contract down before writing code.** It is the only artifact shared between the
   phases. Swagger becomes the contract once the endpoint exists, but you need something to agree
   on first.
3. **Clients are parallel, not sequential.** Once the endpoint is live, cms / web / mobile are
   independent and can be done by different people in any order.

### Which clients does your feature touch?

| Feature kind | core-service | auth | cms | web | mobile |
|---|---|---|---|---|---|
| Admin-managed content that users read | ✅ write + read endpoints | — | ✅ CRUD | ✅ read | ✅ read |
| Admin-only data (orders, reports, RBAC) | ✅ | sometimes | ✅ | — | — |
| Consumer-only (cart, checkout, progress) | ✅ | — | — | ✅ | ✅ |
| Login, OAuth, OTP, password, roles | — | ✅ | ✅ | ✅ | ✅ |
| A new user field | ✅ (owns the column) | ⚠️ read the ownership rule | ✅ | ✅ | ✅ |

---

## Phase 0 — Design the contract

**Do this before opening an editor.** Output is a short spec you can paste into the ticket.

- [ ] **Name the endpoints.** Path, method, audience. core-service groups by audience with path
      segments: `admin/…` for back-office, `web/…` for consumer-authenticated, `integration/t3/…`
      for partners, unprefixed for public/shared. Pick the one that matches who calls it.
- [ ] **Decide public vs permissioned per endpoint.** In core-service this is not optional — every
      route needs `@Public()` or `@Permissions(...)` or it returns `403` before your code runs. If
      permissioned, name the permission strings now (`<resource>:<action>`).
- [ ] **Write the request shape.** Remember `forbidNonWhitelisted: true` — any key a client sends
      that is not in the DTO is a hard `400`. Over-specify rather than under-specify.
- [ ] **Write the response shape,** including the envelope you will actually produce. Decide now
      whether the list key is `items` (do this) or something feature-specific (the tree is
      inconsistent; do not add to the mess).
- [ ] **Pick the pagination style** — `page`+`limit` unless you have a reason. State the exact
      metadata keys.
- [ ] **Decide if it is market-scoped.** If yes, it depends on the `w-country` header and you need
      the controller in the middleware list.
- [ ] **List the i18n message keys** for every user-facing success and failure message.

See [`02-api-contract.md`](02-api-contract.md) for the conventions each of these must follow.

---

## Phase 1 — core-service

Full guide with a worked example: [`03-guide-core-service.md`](03-guide-core-service.md).

- [ ] Domain layer — entity, value objects, repository port (+ its DI `Symbol`), exceptions
- [ ] Application layer — one folder per use case: command/query, result, use case, **spec**
- [ ] Infrastructure — Drizzle table schema, repository adapter, DI module
- [ ] Presentation — controller, HTTP DTOs with `class-validator` + Swagger examples, presentation module
- [ ] **Register in `app.module.ts` twice** — `imports[]` *and* `configure().forRoutes(...)`
- [ ] **Register the table in `schemas/index.ts`** — two edits, import *and* default-export entry
- [ ] **Add permissions** to `shared/constants/permissions.ts` and to
      `scripts/seed-permissions-per-role.ts`, then run `npm run seed:permissions`
- [ ] **Add i18n keys to all 8 locale files**
- [ ] `npm run db:generate`, **read the generated SQL**, then `npm run db:migrate`
- [ ] `npm run lint && npm test`
- [ ] Verify in Swagger at `/api/v1/coreservice/docs`, then deploy to staging

The four registration steps in bold are the ones that fail *silently*. They are the single biggest
source of "it should work but doesn't" on this project — see
[`03-guide-core-service.md`](03-guide-core-service.md#step-9--the-four-registrations-that-fail-silently).

---

## Phase 2 — auth-service (only if needed)

Full guide: [`04-guide-auth-service.md`](04-guide-auth-service.md).

Most features never touch auth. You need it only for credentials, OAuth, OTP, password reset, or
the RBAC tables themselves. **If you are adding a domain field to a user, that is core-service's
job** — see the ownership rule in [`../README.md`](../README.md#51-a-shared-postgresql-database--and-a-rule-you-must-not-break).

- [ ] TypeBox request/response schema in `src/schemas/`
- [ ] Route in `auth.controller.ts` or `rbac.controller.ts`, with `preHandler` guards
- [ ] Business logic in `src/services/`
- [ ] i18n keys in `src/locales/{en,vi}/{errors,success}.json`
- [ ] If you changed a JWT claim, **every client that decodes the token** needs checking
- [ ] `npm test`; verify at `/api/v1/auth/docs`; deploy

---

## Phase 3 — the clients

Each client is independent. Pick the guides you need:

| Client | Guide | Worked example |
|---|---|---|
| Admin back-office | [`05-guide-cms.md`](05-guide-cms.md) | blog posts — list + create + edit |
| Consumer web | [`06-guide-web.md`](06-guide-web.md) | recruit — list + detail + apply form |
| Mobile | [`07-guide-mobile.md`](07-guide-mobile.md) | blog — list + detail |

The pattern is the same in all three even though no code is shared: **declare the endpoint → type
the response → wrap it in a data-fetching layer → render → register the route → register the menu
entry → add translations.** What differs is how many registries you have to touch, and each client
has registries that fail silently if skipped.

### The common cross-client checklist

- [ ] Types written to match the **actual** response payload — log it once, don't trust the ticket
- [ ] Endpoint declared in that client's conventional place
- [ ] Route registered (**and** its permission map, for cms)
- [ ] Menu / navigation entry added, or the page is unreachable by clicking
- [ ] Translations added to **every** locale file (9 for web, 9 for mobile, none for cms)
- [ ] Loading, empty **and** error states handled — not just the happy path
- [ ] No duplicate error toast (cms and web fire one automatically)
- [ ] That repo's verification command passes — see the table in [`../README.md`](../README.md#8-conventions)

---

## Where things go wrong

Ranked by how much time each costs, from experience visible in the codebase.

| Symptom | Cause | Fix |
|---|---|---|
| New endpoint returns `403`, code never runs | No `@Public()` / `@Permissions()` | Deny-by-default guard. Add one. |
| Endpoint returns `401` for a valid token | Controller missing from `app.module.ts` `forRoutes(...)` | `req.user` was never populated |
| Permission granted but still `403` | Permissions are baked into the JWT at login | User must re-login |
| Client gets `400` on a valid-looking body | `forbidNonWhitelisted: true` | Remove the extra key, or add it to the DTO |
| Client reads `data.x` and gets `undefined` | The `data.data` nesting from `return { message, data }` | Log the payload; see [`02-api-contract.md`](02-api-contract.md#the-datadata-trap) |
| Relational query fails at runtime, migration was fine | Table missing from `schemas/index.ts` | drizzle-kit globs files; the runtime uses the barrel |
| Response `message` is the literal `common.foo.bar` | Missing i18n key | Add it to all 8 locale files |
| Page unreachable though the route works | No sidebar / navigation entry | Add it |
| Queue or event feature does nothing, no error | `RABBITMQ_URL` unset | Everything silently no-ops |
| Mobile: fresh clone will not compile | Codegen output is gitignored | `make setup`, then `make gen` after every pull |
| Mobile: app talks to nothing | `flutter run` without `--dart-define-from-file` | Use `make run ENV=dev` |

---

## The documents in this folder

| | |
|---|---|
| [`01-architecture.md`](01-architecture.md) | How each project is structured; core-service's request lifecycle |
| [`02-api-contract.md`](02-api-contract.md) | Envelope, error shapes, pagination, auth, headers — the shared contract |
| [`03-guide-core-service.md`](03-guide-core-service.md) | NestJS hexagonal feature guide |
| [`04-guide-auth-service.md`](04-guide-auth-service.md) | Fastify auth + RBAC guide |
| [`05-guide-cms.md`](05-guide-cms.md) | React admin CRUD guide |
| [`06-guide-web.md`](06-guide-web.md) | Next.js storefront guide |
| [`07-guide-mobile.md`](07-guide-mobile.md) | Flutter feature guide |
| [`08-walkthrough-new-feature.md`](08-walkthrough-new-feature.md) | One feature traced through all five repos |
| [`09-local-setup.md`](09-local-setup.md) | Running everything locally |
