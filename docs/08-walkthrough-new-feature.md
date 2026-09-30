# Walkthrough: one feature across all five repos

A single small feature — **Announcement**: a banner notice that admins manage in the CMS and users
read on web and mobile — traced from contract to release. It is deliberately simple so the *seams*
between repos stay visible; that is what this document teaches.

Every file path below is real, and every pattern comes from the existing code. Follow the linked
guide for the detail of each phase; this document is the spine.

---

## The feature

| | |
|---|---|
| **Entity** | `Announcement` — title, body, market, status, start/end date |
| **Admin** | full CRUD in the CMS, permission-gated |
| **Consumers** | web and mobile read the currently-active announcement for their market |
| **auth-service** | not touched |

---

## Phase 0 — The contract

Written before any code, agreed with whoever will build the clients, pasted into the ticket.

### Endpoints

| Method | Path | Auth | Purpose |
|---|---|---|---|
| `GET` | `/api/v1/coreservice/announcements/list` | `@Public()` | paginated list (CMS) |
| `GET` | `/api/v1/coreservice/announcements/active` | `@Public()` | the one live announcement for the caller's market |
| `GET` | `/api/v1/coreservice/announcements/:id` | `@Public()` | detail (CMS edit form) |
| `POST` | `/api/v1/coreservice/announcements` | `announcement:create` | create |
| `PATCH` | `/api/v1/coreservice/announcements/:id` | `announcement:update` | update |
| `DELETE` | `/api/v1/coreservice/announcements/:id` | `announcement:delete` | delete |

Reads are public because the banner must render for anonymous visitors. Writes are permissioned.

### Request — create / update

```jsonc
{
  "title": "Nghỉ lễ 30/4",
  "body": "<p>Hệ thống tạm dừng…</p>",
  "market": "vn",
  "status": "active",              // active | inactive
  "startDate": "2026-04-28T00:00:00.000Z",
  "endDate": "2026-05-02T00:00:00.000Z"
}
```

`PATCH` takes the same fields, all optional. **No other keys** — the backend runs
`forbidNonWhitelisted: true`, so an extra field is a hard `400`.

### Response — list

```jsonc
{
  "success": true,
  "code": 200,
  "message": "Success",
  "data": {
    "items": [ { "id": "uuid", "title": "…", /* … */ "createdAt": "…", "updatedAt": "…" } ],
    "pagination": { "total": 12, "page": 1, "limit": 20, "totalPages": 1 }
  }
}
```

### Response — active

```jsonc
{ "success": true, "code": 200, "message": "Success", "data": { "announcement": { … } } }
```
`data.announcement` is `null` when nothing is live. **Clients must handle null** — that is the normal
case, not an error.

### Decisions recorded

- Pagination: `page` + `limit`, response `items` + `pagination` — the conventional choice
- Market-scoped: **yes.** `/active` reads `req.market`, so the controller must be in the middleware list
- Permissions: `announcement:create`, `announcement:update`, `announcement:delete`. No `:read`
- i18n keys: `announcement.{create,update,delete}_{success,failed}`
- Envelope: mutations return `{ message, data }`, so **`data` will be nested as `data.data`**.
  Recorded here so no client developer is surprised → [`02-api-contract.md`](02-api-contract.md#the-datadata-trap)

---

## Phase 1 — core-service

Guide: [`03-guide-core-service.md`](03-guide-core-service.md). Copy the `blog-category` file set.

### Files to create

```
src/domain/announcement/
├── entities/announcement.entity.ts
├── value-objects/announcement-id.vo.ts
├── ports/repositories/announcement.repository.port.ts   # + ANNOUNCEMENT_REPOSITORY Symbol
└── exceptions/announcement-not-found.exception.ts

src/application/announcement/use-cases/
├── create-announcement/{…command,…result,…use-case,…use-case.spec}.ts
├── update-announcement/{…command,…result,…use-case,…use-case.spec}.ts
├── delete-announcement/{…command,…use-case,…use-case.spec}.ts
├── get-announcement/{…query,…result,…use-case,…use-case.spec}.ts
├── list-announcements/{…query,…result,…use-case,…use-case.spec}.ts
└── get-active-announcement/{…query,…result,…use-case,…use-case.spec}.ts

src/infrastructure/
├── databases/schemas/announcement.schema.ts
├── persistence/drizzle/repositories/announcement.repository.adapter.ts
└── di/announcement.module.ts

src/presentation/http/rest/announcement/
├── announcement.controller.ts
├── announcement.module.ts
└── dtos/{create,update,list}-announcement-http.dto.ts
```

### The table

```ts
// src/infrastructure/databases/schemas/announcement.schema.ts
const timestamps = {
  createdAt: timestamp('created_at').defaultNow(),
  updatedAt: timestamp('updated_at').defaultNow().$onUpdate(() => new Date()),
};

export const announcements = pgTable('announcements', {
  id: uuid('id').primaryKey().defaultRandom(),
  title: varchar('title', { length: 256 }).notNull(),
  body: text('body').notNull(),
  market: varchar('market', { length: 8 }).notNull(),
  status: varchar('status', { length: 20 }).notNull().default('active'),
  startDate: timestamp('start_date').notNull(),
  endDate: timestamp('end_date').notNull(),
  ...timestamps,
});
```

### The one interesting use case

Everything else is `blog-category` with the names changed. `get-active-announcement` is the one that
carries real logic, and it is market-aware:

```ts
@Injectable()
export class GetActiveAnnouncementUseCase {
  constructor(
    @Inject(ANNOUNCEMENT_REPOSITORY)
    private readonly repository: IAnnouncementRepositoryPort,
  ) {}

  async execute(query: GetActiveAnnouncementQuery): Promise<GetActiveAnnouncementResult> {
    const announcement = await this.repository.findActiveForMarket(query.market, new Date());
    return GetActiveAnnouncementResult.fromEntity(announcement);   // handles null
  }
}
```

The controller reads the market off the request — which only works because of the middleware
registration in the next step:

```ts
@Public()
@Get('active')
@HttpCode(HttpStatus.OK)
@ApiOperation({ summary: 'Get the currently active announcement for the caller market' })
@ApiResponse({
  status: HttpStatus.OK,
  description: 'Active announcement, or null',
  schema: { example: { success: true, code: 200, message: 'Success',
                       data: { announcement: null } } },
})
async getActive(@Req() req: Request & { market?: string }) {
  return this.getActiveUseCase.execute(new GetActiveAnnouncementQuery(req.market ?? 'vn'));
}
```

### The registrations — do not skip these

```ts
// 1. src/app.module.ts — imports[]
AnnouncementPresentationModule,

// 2. src/app.module.ts — configure().forRoutes(...)
AnnouncementController,        // ← without this, req.market is undefined and /active breaks

// 3. src/infrastructure/databases/schemas/index.ts — TWO edits
import * as announcements from './announcement.schema';
export default { /* … */ announcements };

// 4. src/shared/constants/permissions.ts
export const ANNOUNCEMENT_CREATE = 'announcement:create';
export const ANNOUNCEMENT_UPDATE = 'announcement:update';
export const ANNOUNCEMENT_DELETE = 'announcement:delete';
```

Then the seed and the translations:

```bash
# add rows + role grants to scripts/seed-permissions-per-role.ts first
npm run seed:permissions
```

```jsonc
// src/infrastructure/i18n/locales/{en,vi,ms,tl,pt,id,es,th}/common.json — ALL EIGHT
"announcement": {
  "create_success": "Announcement created successfully",
  "create_failed": "Failed to create announcement",
  "update_success": "Announcement updated successfully",
  "update_failed": "Failed to update announcement",
  "delete_success": "Announcement deleted successfully",
  "delete_failed": "Failed to delete announcement"
}
```

### Migrate and verify

```bash
npm run db:generate
# READ src/infrastructure/databases/migrations/NNNN_*.sql
npm run db:migrate
npm run lint && npm test
```

Then at `http://localhost:3435/api/v1/coreservice/docs`:

- `GET /announcements/active` with no token → `200`, `data.announcement` is `null`
- `POST /announcements` with no token → `401`
- `POST` with a token lacking `announcement:create` → `403`
- `POST` with an extra `"foo": 1` in the body → `400` (proves whitelisting works)
- `GET /announcements/active` with `w-country: th` → the Thai market's row, or null

### ✅ Handoff gate

**Deployed to staging, and the response shape confirmed against the Phase 0 contract.** Post the
Swagger link. Clients start now, not before.

> A permission was added, so **everyone testing the CMS must log out and back in** — permissions are
> baked into the 30-minute access token.

---

## Phase 2 — auth-service

**Not touched.** `Announcement` is domain data, so it belongs entirely to core-service, and no JWT
claim changed. See [`04-guide-auth-service.md`](04-guide-auth-service.md#1-first-decide-whether-you-need-it-at-all).

---

## Phase 3a — CMS: the admin CRUD

Guide: [`05-guide-cms.md`](05-guide-cms.md). Copy `src/module/blog/`.

### The service

```ts
// src/services/announcementService.ts
import { cmsClient } from '@/core';
import type { IResponse, ExtendedAxiosRequestConfig } from '@/shared/types';
import type { IAnnouncementFilter, IAnnouncementCreate, IAnnouncementUpdate }
  from '@/module/announcement/types';

class AnnouncementService {
  async list(params: IAnnouncementFilter): Promise<IResponse> {
    return await cmsClient.get('coreservice/announcements/list', { params });
  }
  async getById(id: string): Promise<IResponse> {
    return await cmsClient.get(`coreservice/announcements/${id}`);
  }
  async create(payload: IAnnouncementCreate): Promise<IResponse> {
    return await cmsClient.post('coreservice/announcements', payload);
  }
  async update(id: string, payload: Partial<IAnnouncementUpdate>): Promise<IResponse> {
    return await cmsClient.patch(`coreservice/announcements/${id}`, payload);
  }
  async delete(id: string): Promise<IResponse> {
    return await cmsClient.delete(`coreservice/announcements/${id}`, {
      params: {},
      offNotify: true,
    } as ExtendedAxiosRequestConfig);
  }
}

export const announcementService = new AnnouncementService();
```

Add it to `src/services/index.ts`.

### The types — note the two date shapes

```ts
// src/module/announcement/types/index.ts
export enum AnnouncementStatus { ACTIVE = 'active', INACTIVE = 'inactive' }

export interface IAnnouncement {
  id: string; title: string; body: string;
  market: MarketType; status: AnnouncementStatus;
  startDate: string; endDate: string;          // ISO strings from the API
  createdAt: string; updatedAt: string;
}

export interface IAnnouncementFilter { page: number; limit: number; market?: string; status?: string }
export interface IAnnouncementCreate { /* … */ startDate: string; endDate: string }   // ISO
export interface IAnnouncementUpdate extends IAnnouncementCreate {}
export interface IAnnouncementFormValues {
  title: string; body: string; market: MarketType; status: AnnouncementStatus;
  dateRange: any;                              // dayjs[] from antd RangePicker
}
```

The form/API split is the point — the submit handler converts in one place:

```ts
const handleSubmit = async (values: IAnnouncementFormValues) => {
  const [start, end] = values.dateRange ?? [];
  const payload: IAnnouncementCreate = {
    title: values.title,
    body: values.body,
    market: values.market,
    status: values.status,
    startDate: dayjs(start).toISOString(),
    endDate: dayjs(end).toISOString(),
  };
  const { data, error } = await announcementService.create(payload);
  const created = data?.data?.announcement;        // ⚠ data.data — the Phase 0 contract said so
  if (error || !created?.id) return;
  message.success('Tạo thông báo thành công');
  navigate(ROUTE_NAME.ANNOUNCEMENTS);
};
```

**That `data?.data?.announcement` is why Phase 0 recorded the envelope decision.** Without it you
write `data?.announcement`, get `undefined`, and spend twenty minutes on it.

### The five registrations

```ts
// 1. src/shared/constants/route.ts
ANNOUNCEMENTS: '/announcements',
ANNOUNCEMENT_CREATE: '/announcements/create',
ANNOUNCEMENT_EDIT: '/announcements/edit/:announcement-id',

// 2. src/shared/constants/permissions.ts
ANNOUNCEMENTS: 'announcement',

// 3. src/module/announcement/route.tsx — lazy() + RouteObject[]

// 4. src/router/routes.tsx
...announcementRoutes,

// 5. src/shared/helpers/permission.ts
[ROUTE_NAME.ANNOUNCEMENTS]: PERMISSIONS.ANNOUNCEMENTS,
[ROUTE_NAME.ANNOUNCEMENT_CREATE]: PERMISSIONS.ANNOUNCEMENTS,
[ROUTE_NAME.ANNOUNCEMENT_EDIT]: PERMISSIONS.ANNOUNCEMENTS,

// 6. src/layout/components/SideBar.tsx — an AllMenuItems entry with requiredPermission
```

Miss #5 and users get an unexplainable 403. Miss #6 and the page is unreachable by clicking.

```bash
npm run type-check && npm run lint
```

Verify in the browser, **including as a non-admin without `announcement:*`** — the sidebar entry must
be hidden and the direct URL must 403.

---

## Phase 3b — Web: read the banner

Guide: [`06-guide-web.md`](06-guide-web.md).

```ts
// apps/web/src/services/announcement.api.ts
import { ebookHttpPublic } from '@/core';
import type { ResponseType } from '@/shared/types/http-config.type';
import type { IActiveAnnouncementResponse } from '@/modules/announcement/types';

const PREFIX = 'coreservice/announcements';

class AnnouncementService {
  async getActive(): Promise<ResponseType<IActiveAnnouncementResponse>> {
    return ebookHttpPublic.get(`${PREFIX}/active`);
  }
}
export const announcementApi = new AnnouncementService();
```

`ebookHttpPublic` attaches `w-country` automatically — **which is the entire reason the banner is
market-correct.** A raw `fetch` here would always return the default market.

```ts
// apps/web/src/shared/queries/useAnnouncementQueries.tsx
export const useActiveAnnouncement = () =>
  useQuery<IActiveAnnouncementResponse>({
    queryKey: [QUERY_KEYS.ANNOUNCEMENT.ACTIVE],
    queryFn: async () => {
      const response = await announcementApi.getActive();
      return response.data;
    },
    staleTime: 5 * 60 * 1000,
  });
```

Add `ANNOUNCEMENT: { ACTIVE: 'announcement/active' }` to
`packages/constants/src/tanstack-query.ts`.

The component handles the null case as normal, not as an error:

```tsx
'use client';
export function AnnouncementBanner() {
  const t = useTranslations('Announcement');
  const { data } = useActiveAnnouncement();
  const announcement = data?.announcement;

  if (!announcement) return null;        // ← the common case

  return (
    <div className="bg-primary/10 px-4 py-3" role="status">
      <p className="font-semibold">{announcement.title}</p>
      <div dangerouslySetInnerHTML={{ __html: announcement.body }} />
      <button onClick={handleDismiss} aria-label={t('dismiss')}>×</button>
    </div>
  );
}
```

Because the banner is global, prefetch it in `src/app/[locale]/layout.tsx` next to the existing
category prefetch rather than adding a route. **`Announcement.dismiss` must go into all 9
`messages/*.json` files** — next-intl v4 *throws* on a missing key.

```bash
npm run type-check && npm run lint && npm run lint:css
```

---

## Phase 3c — Mobile: read the banner

Guide: [`07-guide-mobile.md`](07-guide-mobile.md). Copy `lib/features/blog/`, minus the screens.

```dart
// 1. lib/core/network/api_endpoints.dart
static const announcementActive = '/coreservice/announcements/active';

// 2. lib/features/announcement/data/datasources/announcement_remote_data_source.dart
final class AnnouncementRemoteDataSource {
  const AnnouncementRemoteDataSource(this._apiClient);
  static const activeEndpoint = ApiEndpoints.announcementActive;
  final ApiClient _apiClient;

  /// `GET /coreservice/announcements/active`. Public endpoint.
  /// `data.announcement` là null khi không có thông báo nào đang chạy.
  Future<AnnouncementModel?> getActive() async {
    final response = await _apiClient.requestJson<Map<String, dynamic>>(
      activeEndpoint,
      authMode: NetworkAuthMode.none,          // public — must opt out explicitly
    );

    final data = _unwrapData(response.data, 'announcements/active');
    final announcement = data['announcement'];
    if (announcement == null) return null;     // not an error
    if (announcement is! Map) {
      throw const FormatException('announcements/active: data.announcement sai kiểu.');
    }

    try {
      return AnnouncementModel.fromJson(Map<String, dynamic>.from(announcement));
    } on TypeError catch (error) {
      throw FormatException('announcements/active sai kiểu: $error');
    }
  }
}
```

The `w-country` header comes from `MarketInterceptor` on `dioProvider`, derived from **device
region**, not UI language. You do not set it.

```dart
// 3. the Freezed model — every field nullable or defaulted
@freezed
abstract class AnnouncementModel with _$AnnouncementModel {
  const factory AnnouncementModel({
    required String id,
    required String title,
    @Default('') String body,
    @Default('vn') String market,
    @Default('active') String status,
    String? startDate,
    String? endDate,
  }) = _AnnouncementModel;

  factory AnnouncementModel.fromJson(Map<String, dynamic> json) =>
      _$AnnouncementModelFromJson(json);
}

// 4. mapper, 5. entity, 6. repository contract returning Result<Announcement?>,
// 7. repository impl with the _guard mapping

// 8. lib/features/announcement/announcement_providers.dart
final announcementRemoteDataSourceProvider = Provider<AnnouncementRemoteDataSource>((ref) {
  return AnnouncementRemoteDataSource(ref.watch(apiClientProvider));
});
final announcementRepositoryProvider = Provider<AnnouncementRepository>((ref) {
  return AnnouncementRepositoryImpl(ref.watch(announcementRemoteDataSourceProvider));
});

// 9. a FutureProvider — a simple GET with no actions needs no AsyncNotifier
final activeAnnouncementProvider = FutureProvider.autoDispose<Announcement?>((ref) async {
  final result = await ref.watch(announcementRepositoryProvider).getActive();
  return result.fold(onSuccess: (value) => value, onFailure: (_) => null);
});
```

Swallowing the failure to `null` is a deliberate choice **for a non-essential banner** — a dead
announcement endpoint must not break the home screen. For anything the user asked for, throw so the
screen can show an error with retry.

```dart
// 10. the widget
final announcement = ref.watch(activeAnnouncementProvider).valueOrNull;
if (announcement == null) return const SizedBox.shrink();
```

No route to register — it is a widget inside the existing home screen, so `route_names.dart`,
`app_router.dart` and `_tabPaths` are untouched.

**Add `announcementDismiss` to all 9 `.arb` files**, then:

```bash
make gen       # regenerates Freezed + l10n; check_l10n fails on a missing key
make check     # format + analyze + test
```

Verify on **both** an Android emulator and an iOS simulator, and with airplane mode on — the banner
must vanish, not crash the home screen.

---

## What this feature cost, per repo

| Repo | New files | Edited files | The silent-failure risks |
|---|---|---|---|
| core-service | 19 | 5 | `forRoutes` (breaks `/active`), `schemas/index.ts`, permission seed, 8 locale files |
| auth-service | 0 | 0 | — |
| cms | 10 | 6 | `ROUTE_PERMISSION_MAP`, `AllMenuItems` |
| web | 4 | 5 | 9 message files (throws — at least it is loud) |
| mobile | 8 | 3 | 9 ARB files (caught by `check_l10n`), `NetworkAuthMode` |

`auth-service` being empty is the normal case. Most features never touch it.

---

## The seams — what this walkthrough was really about

**1. The contract is the only artifact that crosses a repo boundary.** Phase 0 recorded that
mutations return a nested `data.data`, and that one line saved debugging time in the CMS. Nothing
else in the five repos would have told you.

**2. Deploy before the clients start.** Three developers could have written Phase 3a, 3b and 3c in
parallel — but only against a live staging endpoint. Against an imagined one, all three would have
guessed the `data.announcement` null-handling differently.

**3. Silent registration failures are the dominant bug class.** Nine of the twelve edited files
across the five repos fail *quietly*: the route responds, the page renders, the build passes, and the
feature is subtly broken. `forRoutes` is the worst — `/active` would return the default market
forever and look like a backend bug.

**4. The same idea is spelled differently everywhere.** `items` in core became `data.items` in the
CMS hook, `response.data` in a TanStack query, and `_unwrapData(…)['announcement']` in Dart. Nothing
is shared, and nothing checks that they agree. Only the contract document and a live endpoint do.

**5. Permission changes need a re-login.** Adding `announcement:create` did nothing for anyone
already logged in until their token expired. Say so in the PR description.
