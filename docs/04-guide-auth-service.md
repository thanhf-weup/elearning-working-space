# Changing auth-service

`elearning-authentication-service` is small and deliberately different from core-service: Fastify
instead of NestJS, TypeBox instead of class-validator, module singletons instead of DI, no
repository layer. Do not carry core-service habits across.

---

## 1. First decide whether you need it at all

Most features never touch this service.

| Change | Where it belongs |
|---|---|
| Login, signup, logout, refresh | **auth-service** |
| Google OAuth (web or mobile) | **auth-service** |
| OTP, forgot/reset password | **auth-service** |
| Creating a role or permission, assigning roles | **auth-service** (it owns the RBAC tables) |
| A claim added to the JWT | **auth-service** — and then every client that decodes it |
| A new profile field on a user | **core-service** — it owns `users` beyond the auth columns |
| Anything about products, orders, content, learning | **core-service** |
| A new permission *string* consumed by a core endpoint | **core-service** constants + seed, then granted here |

> ### ⚠️ The ownership rule
> auth-service owns exactly seven tables: `users`, `authentications`, `roles`, `permissions`,
> `role_permissions`, `user_roles`, `password_reset_tokens`. core-service owns the other 61 — and it
> has **already added `status`, `deleted_at` and `deleted_by` to `users`**, which auth's schema file
> does not know about.
>
> Running `npm run db:generate` here right now will emit
> `ALTER TABLE users DROP COLUMN status, DROP COLUMN deleted_at, DROP COLUMN deleted_by` and destroy
> core-service's user-soft-delete feature. **Read the generated SQL, every time.** Full explanation
> in [`../README.md`](../README.md#51-a-shared-postgresql-database--and-a-rule-you-must-not-break).

---

## 2. Getting it running

```bash
cd elearning-authentication-service
npm ci
cp .env.example .env
npm run dev          # tsx watch src/index.ts → http://localhost:8000
```

**That crashes.** `src/config/vars.config.ts` validates env at import time and throws on five
variables that `.env.example` does not contain. The first error is `Error: API_URL is not defined`,
raised before Fastify starts because `swagger.config.ts` imports `vars.config` at module scope.

The eleven variables that are actually required:

```bash
FE_URL=http://localhost:3000
GOOGLE_OAUTH2_REDIRECT_PATH=/api/v1/auth/login/google
GOOGLE_OAUTH2_CALLBACK_PATH=/api/v1/auth/login/google/callback
GOOGLE_CLIENT_ID=…
GOOGLE_CLIENT_SECRET=…
GOOGLE_IOS_CLIENT_ID=…
API_URL=http://localhost:8000          # ← missing from .env.example
MAILJET_API_KEY=…                      # ← missing from .env.example
MAILJET_API_SECRET=…                   # ← missing from .env.example
MAILJET_FROM_EMAIL=…                   # ← missing from .env.example
MAILJET_FROM_NAME=…                    # ← missing from .env.example
```

Plus these, which have defaults or fail quietly:

```bash
POSTGRES_URL=postgres://…              # required in practice
JWT_ACCESS_TOKEN_SECRET_KEY=…
JWT_REFRESH_TOKEN_SECRET_KEY=…
SESSION_SECRET=…
API_PORT=8000
CORE_SERVICE_BASE_URL=http://localhost:3435   # optional — tracking silently no-ops without it
RABBITMQ_URL=amqp://…                         # optional — publishes silently no-op without it
KONG_JWT_ISS=…                                # in no example file; becomes the JWT `iss` claim
UPSTASH_REDIS_URL=…                           # falls back to an in-memory Map
COOKIE_SECRET=…                               # defaults to the literal 'supersecret'
```

All scripts:

```
npm run dev            tsx watch src/index.ts
npm run build          tsc -p tsconfig.json
npm start              node dist/index.js
npm test               vitest run
npm run format         prettier --write .
npm run db:generate    drizzle-kit generate
npm run db:migrate     drizzle-kit migrate
npm run db:seed        tsx src/db/seeds/rbac-seed.ts
```

There is **no lint script**. Swagger is at `http://localhost:8000/api/v1/auth/docs`.

---

## 3. Adding an endpoint

Four files at most, and one of them is optional.

### Step 1 — the TypeBox schema

```ts
// src/schemas/auth.schema.ts
import { Type } from '@sinclair/typebox';

export const LoginSchema = Type.Object({
  email: Type.String({ format: 'email' }),
  password: Type.String({ minLength: 8 }),
});
```

Response envelopes are shared. `src/schemas/common.schema.ts` exports `createRouteSchema()`, which
attaches the standard `200/400/401/500` responses so you only declare the request:

```ts
const createRouteSchema = (options: any = {}) => {
  const { tags, body, querystring, params, consumes, response, ...rest } = options;
  return {
    tags,
    ...(body && { body }),
    ...(querystring && { querystring }),
    ...(params && { params }),
    response: {
      200: SuccessResponse,
      400: BadRequestResponse,
      401: UnauthorizedResponse,
      500: InternalServerErrorResponse,
      ...response,        // override when you need a 409 etc.
    },
    ...rest,
  };
};
```

`SuccessResponse` is `{ success: true, data: any, message?: string, metadata?: any }` — note
`metadata`, where core-service would nest `pagination` inside `data`.

There is also `PaginationQuerySchema` (**default `limit: 10`, not 20 like core**),
`PaginationCursorQuerySchema` and `IdParamsSchema`.

### Step 2 — the route

Routes are registered as Fastify plugin functions. There is no decorator layer.

```ts
// src/controllers/auth.controller.ts
export const AuthController = async (route: FastifyInstance) => {
  route.post(
    '/login',
    {
      schema: createRouteSchema({
        tags: ['Auth'],
        body: LoginSchema,
      }),
    },
    async (request: FastifyRequest, reply: FastifyReply) => {
      const { email, password } = request.body as { email: string; password: string };

      if (!/^(?=.*[a-zA-Z])(?=.*\d).{8,}$/.test(password)) {
        return throwError('passwordInvalidFormat', 400);
      }

      const data = await authService.login(email, password, { ip, userAgent });
      return successHandler(request, reply, data, null, {
        translationKey: 'loginSuccess',
      });
    },
  );
};
```

`src/index.ts` is the **only** place plugins are mounted:

```ts
app.register(AuthController, { prefix: 'api/v1/auth' });
app.register(rbacController, { prefix: 'api/v1/auth/rbac' });
```

So a route declared as `/login` is served at `/api/v1/auth/login`.

### Step 3 — guards, if the route is not public

Per-route `preHandler`, not a global guard. **Two styles exist in the tree** — match the file you
are editing:

```ts
// rbac.controller.ts — array form
{
  preHandler: [authenticateToken, requirePermission('role', 'create')],
  schema: createRouteSchema({ /* … */ }),
}

// auth.controller.ts — sequential-await form
{
  preHandler: async (request, reply) => {
    await authenticateToken(request, reply);
    await requirePermission('user', 'create')(request, reply);
  },
}
```

`authenticateToken` (`src/middlewares/auth.middleware.ts`) does real verification — unlike
core-service:

```ts
// Verify the token
const decoded = jwt.verify(
  token,
  process.env.JWT_ACCESS_TOKEN_SECRET_KEY,
) as JWTPayload;

// Attach user info to request
(request as any).user = decoded;
```

It accepts `Authorization: Bearer <t>` first, then the `access_token` cookie, and distinguishes
`TokenExpiredError` → `accessTokenExpired` from other failures → `invalidAccessToken`, both `401`.

`requirePermission(resource, action)` duplicates core's wildcard matching logic verbatim.

### Step 4 — business logic

Services are plain module singletons in `src/services/`, talking to Drizzle directly. There is no
repository layer and no DI container:

```ts
import db from '../db';
// …
export default { login, signup, /* … */ };
```

`auth.service.ts` is 658 lines and holds most of the logic. `login()` does: case-insensitive email
lookup → Redis blacklist check → bcrypt compare → update `users.lastLoginAt` →
`Promise.all([getUserPermissions, getUserRoles])` → `generateToken` → fire-and-forget
`publishUserActivity`.

---

## 4. Responses and errors

**Success** — call `successHandler`, which always replies `200`:

```ts
return successHandler(request, reply, data, metadata, { translationKey: 'loginSuccess' });
// → { success: true, data, metadata?, message }
```

**Errors** — `throwError` builds a plain `Error` carrying a status and a translation key:

```ts
const throwError = (translationKey: string, statusCode: number = 500, interpolationParams?: any) => {
  const error = new Error(translationKey) as Error & { statusCode: number; translationKey: string };
  error.statusCode = statusCode;
  error.translationKey = translationKey;
  error.interpolationParams = interpolationParams;
  throw error;
};
```

A single `app.setErrorHandler(errorHandler)` translates it and replies:

```jsonc
{ "success": false, "data": null, "message": "…", "code": 401 }
```

This matches core-service's `I18nExceptionFilter` shape, but not `AllExceptionsFilter`'s.

---

## 5. i18n

`i18next` + `i18next-fs-backend`, loaded from `process.cwd()/src/locales/{{lng}}/{{ns}}.json`.

**Only two locales and two namespaces actually exist:**

```
src/locales/{en,vi}/{errors,success}.json
```

`i18n.config.ts` declares four namespaces (`common`, `errors`, `auth`, `success`) but files exist for
only two — `common` and `auth` resolve to nothing. `defaultNS` is `errors`, which is why
`throwError('passwordInvalidFormat', 400)` needs no namespace prefix.

Two gotchas:

- **The config module exports an already-invoked promise** (`export default initI18next()`), and
  `index.ts` must `await` it before registering the middleware.
- **Language comes from `accept-language` only** — parsed as
  `acceptLanguage.split(',')[0].split('-')[0]`. Unlike core-service, auth does **not** honour
  `x-lang` or `x-locale`.

---

## 6. RBAC tables and seeding

Seven tables in `src/db/schemas/`. Permissions are stored as `resource` + `action` rows and assembled
into `resource:action` strings by SQL joins in `src/utils/rbac.utils.ts` (`userRoles` →
`rolePermissions` → `permissions`).

```bash
npm run db:seed          # src/db/seeds/rbac-seed.ts
```

core-service has its own seeds, which are the ones that create login accounts:

```bash
cd ../elearning-core-service
npm run seed:roles         # roles + 4 users, all with the password Admin@123
npm run seed:permissions   # permission rows + role grants
```

> The seeded accounts (`admin@`, `contentcreator@`, `operator@`, `operatormanager@weupbook.com`) all
> share the hardcoded password **`Admin@123`** and the `*:*` permission. Change them before any
> shared environment — `npm run change:password` exists for this.

---

## 7. Tokens — and what changing them costs

```ts
// src/utils/generateToken.util.ts
accessTokenPayload['iss'] = process.env.KONG_JWT_ISS;
const accessToken  = jwt.sign(accessTokenPayload, process.env.JWT_ACCESS_TOKEN_SECRET_KEY,  { expiresIn: '30m' });
const refreshToken = jwt.sign(payload,            process.env.JWT_REFRESH_TOKEN_SECRET_KEY, { expiresIn: '7d'  });
```

HS256 symmetric. Access 30 min, refresh 7 days, password-reset 5 min (signed with the *access*
secret).

> **Changing the JWT payload is a breaking change across four clients.** The claims are consumed by:
> - core-service — `IJwtPayload`, read by the middleware and the permissions guard
> - cms — `DecodedToken` in `src/shared/types/role.ts`, which drives the entire RBAC menu and route gating
> - web — `decodeAccessToken` in `src/lib/token-manager.ts`
> - mobile — `lib/core/network/jwt_utils.dart`
>
> Note core-service declares the payload contract twice and inconsistently:
> `shared/constants/user-request.type.ts` has `role?: string` (singular) while
> `application/dtos/types/jwt-payload.interface.ts` has `roles`. Check both.

**Revocation.** `POST /logout` decodes (does not verify) the token and publishes a `user_logout`
activity — it does **not** write the blacklist. Blacklist entries (`blacklist:user:<userId>`) are
written by the user-deletion flow and read by both services. That is the only mechanism that revokes
access mid-token-lifetime.

---

## 8. Verify

```bash
npm test      # vitest — only 2 test files exist: tests/services/{auth,mail}.service.test.ts
```

Note `tsconfig.json` **excludes** `tests` and `**/*.test.ts`, so test files are not type-checked by
`npm run build`. There is no lint script and no CI workflow in this repo at all.

Then exercise it manually at `http://localhost:8000/api/v1/auth/docs`:

- login with valid credentials → token pair, with `permissions` and `roles` populated
- login with a wrong password → `401` and a translated message
- a permissioned RBAC route without a token → `401`
- the same route with a token lacking the permission → `403`
- `Accept-Language: vi` → a Vietnamese message

---

## Checklist

```
[ ] Confirmed this belongs in auth-service, not core-service
[ ] TypeBox schema in src/schemas/
[ ] Route registered in auth.controller.ts or rbac.controller.ts
[ ] Prefix confirmed — index.ts mounts api/v1/auth and api/v1/auth/rbac
[ ] preHandler guards added if not public, matching the file's existing style
[ ] Logic in src/services/, using the db singleton
[ ] successHandler for success, throwError for failure
[ ] i18n keys in src/locales/{en,vi}/{errors,success}.json
[ ] If a JWT claim changed: all four clients checked
[ ] If a table changed: ownership rule respected, generated SQL read by a human
[ ] npm test
[ ] Exercised in Swagger, including the 401 and 403 paths
```
