# The API contract

Everything all four clients depend on. Read this before designing an endpoint, and again before
writing the client types.

---

## 1. Base URLs and prefixes

One host; the path prefix selects the service.

| Environment | Host |
|---|---|
| Staging | `https://staging.api.weupbook.com` |
| Production | `https://api.weupbook.com` |
| Local core-service | `http://localhost:3435` |
| Local auth-service | `http://localhost:8000` |

| Prefix | Service | Source |
|---|---|---|
| `/api/v1/auth/*` | auth-service | `src/index.ts:48` |
| `/api/v1/auth/rbac/*` | auth-service | `src/index.ts:49` |
| `/api/v1/coreservice/*` | core-service | `src/main.ts:72` |

How each client configures it:

```ts
// cms — src/core/cmsClient.ts
new BaseClient({ baseURL: `${import.meta.env.VITE_API_URL}/api/v1/` })
// then: cmsClient.get('coreservice/blogs/list')   and   cmsClient.post('auth/login')

// web — src/core/utils/ebook-http.ts
new BaseHttp({ baseURL: `${process.env.NEXT_PUBLIC_SERVER_API_URL}/api/v1/` })
```
```dart
// mobile — config/dev.json
"API_BASE_URL": "https://staging.api.weupbook.com/api/v1"
// paths registered centrally in lib/core/network/api_endpoints.dart:
static const blogList = '/coreservice/blogs/list';
```

### Versioning is a literal, not a mechanism

`v1` is hardcoded into the prefix string. Neither service calls `enableVersioning()` and there are
no `@Version()` decorators. **There is no way to ship a v2 of one endpoint.** core-service segments
by *audience* instead:

| Segment | Audience | Auth |
|---|---|---|
| `admin/…` | back-office | JWT + permissions |
| `web/…` | logged-in consumers | JWT |
| `integration/t3/…` | partners | API key guard |
| *(unprefixed)* | public / shared | `@Public()` or JWT |

Two paths escape the global prefix entirely: `/docs` (Swagger) and `/admin/queues` (BullMQ
dashboard). **Both are unauthenticated.**

---

## 2. The success envelope

core-service applies it globally in `ResponseTransformInterceptor`:

```ts
export interface Response<T> {
  success: boolean;
  data: T;
  message: string;
  code: number;      // the HTTP status
}
```

The interceptor has three branches:

1. Your return value already has `success`, `data` and `message` → passed through, `message` translated.
2. Your return value **contains a `message` key** → `message` is *lifted out*, everything else
   becomes `data`. If nothing else remains, `data` is `null`.
3. Anything else → `{ success: true, data: <your value>, message: 'Success', code: <status> }`.

### The `data.data` trap

Branch 2 means the two controller styles in the tree produce **different response shapes**:

```ts
// blog-category style
return { message: i18n.t('common.blogCategory.create_success'), data: result };
// → { success: true, data: { data: result }, message: "…", code: 201 }     ⚠ NESTED

// department style
return { message: i18n.t('…'), ...result };
// → { success: true, data: result, message: "…", code: 201 }               ✅ flat
```

This is why cms's create page reads `createData?.data?.blog` while its list hook reads
`responseData?.items`. **Log the actual payload once before writing client types.** Both styles are
live; do not silently "fix" one, because clients depend on the current shape.

For new endpoints, prefer returning the flat shape or an explicit
`{ success, data, message }` object so branch 1 or 3 applies and nesting cannot surprise anyone.

### auth-service does it differently

No interceptor. Controllers call `successHandler(request, reply, data, metadata, { translationKey })`,
which **always replies `200`**:

```jsonc
{ "success": true, "data": { … }, "metadata": { … }, "message": "…" }
```

Note `metadata` (only present when non-null) where core would use `pagination` inside `data`.

---

## 3. Error responses — two shapes

Both filters are registered globally in core's `main.ts`. Which one you get depends on whether the
throw was an `HttpException`:

```ts
app.useGlobalFilters(app.get(AllExceptionsFilter), app.get(I18nExceptionFilter));
// comment in main.ts: "Order matters: last filter in array executes FIRST"
```

**`I18nExceptionFilter`** — catches `HttpException`, so it handles almost everything you throw
deliberately:

```jsonc
{ "success": false, "data": null, "code": 404, "message": "Resource not found" }
```

**`AllExceptionsFilter`** — the fallback for unhandled errors, raw Postgres errors and
`DomainException`:

```jsonc
{ "success": false, "statusCode": 500, "timestamp": "2026-09-30T…Z",
  "path": "/api/v1/coreservice/…", "message": "…", "details": {} }
```

> Note the incompatibility: **`code` vs `statusCode`**, and `data: null` vs `timestamp`/`path`.
> Client error handling must tolerate both. mobile's `api_failure_mapper.dart` handles three
> envelope variants for exactly this reason.

It also maps raw Postgres error codes:

| PG code | HTTP | Message |
|---|---|---|
| `22P02` | 400 | Invalid input format (a malformed UUID) |
| `23503` | 400 | Referenced entity not found (FK violation) |
| `23505` | 409 | Duplicate entry (unique violation) |
| `XX000` | 503 | Database service temporarily unavailable |

auth-service emits the `I18nExceptionFilter` shape (`{success, data, message, code}`) via
`throwError(translationKey, statusCode, params)`.

### Validation errors

The global `ValidationPipe` in core's `main.ts`:

```ts
new ValidationPipe({
  whitelist: true,
  forbidNonWhitelisted: true,      // ⚠ an unknown body key is a hard 400
  transform: true,
  enableImplicitConversion: true,
  exceptionFactory: (messages) =>
    new BadRequestException({ message: 'Validation failed: ' + messages.join(', '), errors: messages }),
})
```

> ### ⚠️ `forbidNonWhitelisted: true`
> **Sending one key the DTO does not declare is a `400`, not a warning.** This is the most common
> client↔server friction on this project. Clients must send exactly the declared fields — which is
> why cms's hooks build params by conditional assignment rather than passing a whole form object:
> ```ts
> const params: IBlogPostFilter = { offset: filters.offset, limit: filters.limit };
> if (filters.title) params.title = filters.title;      // omit empty keys, never send undefined
> ```

`400` responses pass the raw message through untranslated so class-validator detail survives.

---

## 4. Pagination

**There is no shared helper.** Each of the ~66 list results computes its own `totalPages`, and the
conventions diverge. Check the endpoint, every time.

### Request

| Style | Used by | Params |
|---|---|---|
| Page-based (dominant) | most endpoints | `page` (min 1), `limit` (min 1, max 100, default 20) |
| Offset-based | `blogs/list` and others | `offset` (min 0), `limit` |

The DTO idiom — `@Type(() => Number)` is required because query strings arrive as strings:

```ts
@ApiPropertyOptional({ default: 20, minimum: 1, maximum: 100 })
@IsOptional()
@Type(() => Number)
@IsInt()
@Min(1)
@Max(100)
limit?: number = 20;
```

DTO defaults apply **only when the key is absent**, which is why controllers re-apply them:

```ts
new ListBlogCategoriesQuery(dto.page ?? 1, dto.limit ?? 20, dto.name)
```

### Response

The dominant shape, from `list-blog-categories.result.ts`:

```jsonc
{
  "items": [ … ],
  "pagination": { "total": 42, "page": 1, "limit": 20, "totalPages": 3 }
}
```

But **the item-array key is not standardised**:

| Endpoint | Key |
|---|---|
| `blog-categories/list`, `blogs/list` | `items` |
| `departments` | `departments` |
| `job-descriptions` | `job_descriptions` *(snake_case leaks into JSON)* |

Use `items` for anything new. Also note `Math.ceil(0 / limit) === 0`, so **an empty list reports
`totalPages: 0`**, not 1 — asserted in the blog-category spec, so clients must not assume ≥ 1.

Clients cope by normalising. cms handles both request styles in one hook:

```ts
const { page, offset, limit, total, totalPages } = responseData.pagination;
const currentPage = page ?? (offset !== undefined ? Math.floor(offset / limit) + 1 : 1);
```

---

## 5. Authentication and authorization

### Tokens

Issued by auth-service only, in `src/utils/generateToken.util.ts`:

```ts
const accessTokenPayload = { ...payload };
accessTokenPayload['iss'] = process.env.KONG_JWT_ISS;
const accessToken  = jwt.sign(accessTokenPayload, process.env.JWT_ACCESS_TOKEN_SECRET_KEY,  { expiresIn: '30m' });
const refreshToken = jwt.sign(payload,            process.env.JWT_REFRESH_TOKEN_SECRET_KEY, { expiresIn: '7d'  });
```

- **HS256 symmetric secrets** — not RS256, so there is no public key to distribute.
- Access **30 min**, refresh **7 days**, password-reset **5 min**.
- The payload carries `{ email, authenticationId, userId, permissions: string[], roles: string[], iss }`.

> **The full permissions array is baked into the access token.** Revoking a permission has no effect
> until the token expires (≤ 30 min) or the user re-logs in. The only immediate revocation is the
> Redis key `blacklist:user:<userId>`, checked by both services.

### Verification

| | Verifies signature? |
|---|---|
| auth-service | ✅ `jwt.verify` with HS256 (`src/middlewares/auth.middleware.ts`) |
| core-service | ❌ **decode only** — see [`../README.md`](../README.md#3-security-and-the-trust-boundary) |

core's middleware also treats a malformed token as **anonymous**, not as an error — the `catch`
calls `next()`. So a client bug that corrupts the token produces `403 Forbidden` from the guard
rather than `401 Unauthorized`.

### Permissions

Format `<resource>:<action>`, listed in `shared/constants/permissions.ts`:

```ts
export const BLOG_CATEGORY_CREATE = 'blog-category:create';
export const BLOG_CATEGORY_UPDATE = 'blog-category:update';
export const BLOG_CATEGORY_DELETE = 'blog-category:delete';
export const ALL_PERMISSIONS = '*:*';
```

Matching in `PermissionsGuard.matchesPermission`, in order: exact string, `*:*`, `<resource>:*`,
`*:<action>`.

Three things to internalise:

1. **Deny by default.** No `@Public()` and no `@Permissions()` → `403` before your handler runs.
2. **`@Permissions(a, b)` is OR.** Any one of them suffices. There is no AND.
3. **A permission string must exist as a row in the `permissions` table and be granted to a role**,
   or the JWT will never contain it. Constant + seed script + `npm run seed:permissions`.

Naming is inconsistent in the existing catalogue — `job_description:create` (snake) alongside
`blog-category:create` (kebab). Match the resource's neighbours.

`@Permissions` already applies `@ApiBearerAuth('JWT-auth')`, so adding it again is redundant (though
several controllers do).

### Non-JWT authentication

Five bespoke guards protect machine-to-machine endpoints, each a shared secret or HMAC:

| Guard | Env var |
|---|---|
| `webhook/guards/ladisales-api-key.guard.ts` | `LADISALES_API_KEY` |
| `webhook/guards/pos-webhook-api-key.guard.ts` | `POS_WEBHOOK_KEY` |
| `webhook/guards/shopify-hmac.guard.ts` | needs `req.rawBody`, wired in `main.ts` |
| `integration/guards/t3-api-key.guard.ts` | `T3_INTEGRATION_API_KEY` |
| `modules/upload/guards/ai-storage-api-key.guard.ts` | `AI_STORAGE_API_KEY` |

Controllers behind these are **deliberately absent** from `app.module.ts`'s middleware list, because
they do not use JWTs.

---

## 6. Headers

| Header | Meaning | Who sends it |
|---|---|---|
| `Authorization: Bearer <jwt>` | identity + permissions | all clients |
| `w-country` | market code — drives pricing, catalog, content | web, mobile |
| `accept-language` | response message language | web, mobile |
| `x-lang` / `x-locale` | language, higher precedence than `accept-language` | any (core only) |

core resolves language as: `x-lang` / `x-locale` header → `accept-language` → `?lang=` / `?locale=`
→ cookie → `'en'`. **auth-service reads `accept-language` only.**

Market resolution (`GetMarketMiddleware`): `w-country` → `cf-ipcountry` → `DEFAULT_MARKET` (`vn`),
with a 5-minute in-memory cache of valid codes and `us` as the fallback for an unsupported detected
market. `en` is force-mapped to `us`.

> **Market codes and locale codes are not the same.** The `Market` enum uses `vn`; i18n uses `vi`.
> core's i18n module bridges them with `fallbacks: { vn: 'vi', 'vn-*': 'vi' }`.

Each client sets these automatically in its HTTP layer. **Calling `fetch` directly bypasses them and
silently returns the wrong market's data.**

---

## 7. i18n of response messages

core-service uses `nestjs-i18n` with **8 locales**: `en vi ms tl pt id es th`, one namespace file
each at `src/infrastructure/i18n/locales/<lang>/common.json`.

Key convention is `<feature>.<action>_{success,failed}`:

```jsonc
"blogCategory": {
  "create_success": "Blog category created successfully",
  "create_failed": "Failed to create blog category",
  "update_success": "Blog category updated successfully",
  …
}
```

In a controller the `common.` prefix is required:

```ts
return { message: i18n.t('common.blogCategory.create_success'), data: result };
```

but the interceptor and exception filter auto-prepend `common.` when translating a bare key from a
thrown message. A key is treated as translatable if it contains a `.` and no space.

> **A missing key is not an error.** `i18n.t()` returns the key string verbatim, so a forgotten
> translation ships to users as the literal `common.widget.create_success` in the `message` field.
> Add every new key to **all 8** files. core-service has a helper skill at
> `.claude/skills/check-i18n/SKILL.md`.

Top-level key naming is mixed camelCase (`blogCategory`, `courseModule`) and snake_case
(`promotional_product`, `job_description`). Match the neighbours.

---

## 8. Swagger

| Service | URL |
|---|---|
| core-service | `/api/v1/coreservice/docs` |
| auth-service | `/api/v1/auth/docs` |

Once an endpoint exists, **Swagger is the contract** — it is what client developers will read.
core-service's standing rule is that every new endpoint carries a real `@ApiResponse` with
`schema.example`, and every request body an `@ApiBody` with `examples`. A `@ApiOperation({ summary })`
that just restates the method name is not enough for someone writing a client against it.

---

## 9. Designing a new endpoint — the short checklist

- [ ] Audience segment chosen (`admin/`, `web/`, `integration/`, public)
- [ ] `@Public()` or `@Permissions(...)` on **every** route
- [ ] Permission constants defined **and** added to the seed script
- [ ] Request DTO declares **every** field a client will send (`forbidNonWhitelisted`)
- [ ] Numeric query params carry `@Type(() => Number)`
- [ ] Pagination uses `page`+`limit`, response uses `items`+`pagination`
- [ ] Response shape checked against the interceptor's branches — no accidental `data.data`
- [ ] i18n keys added to all 8 locale files
- [ ] Swagger has real examples, not just a summary
- [ ] Market-scoped? Then the controller must be in `app.module.ts`'s middleware list
