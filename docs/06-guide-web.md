# Adding a page to the consumer web app

`elearning-fe` is a Turborepo monorepo. The storefront is `apps/web` (Next.js 16 App Router, React
19, Ant Design 6, Tailwind 4, TanStack Query, Zustand, next-intl). Worked through **recruit**
(`src/modules/recruit/`) — a list page, a detail page and a submit form.

Two facts that cause the most confusion here, stated up front:

- **The middleware file is `apps/web/src/proxy.ts`**, not `middleware.ts`. Next.js 16 renamed it.
- **`src/shared/` is a re-export bridge, not a home for new code.** Genuinely shared things go in
  `packages/`.

Existing docs: `elearning-fe/README.md` (conventions), `HANDOFF.md` (honest status, including "no
test infra"), `.github/instructions/intruction-copilot.instructions.md`.

---

## 1. Getting it running

```bash
cd elearning-fe
npm ci                # ⚠ from the REPO ROOT only — never inside apps/web
npm run dev:web       # → http://localhost:3000
```

> **Installing inside `apps/web` breaks hoisting.** The `dev` script copies
> `../../node_modules/pdfjs-dist/legacy/build/pdf.worker.min.mjs` into `public/`, and that path only
> resolves with a root install. Without it the PDF reader dies with "PDF information is not
> available".

Root scripts (turbo orchestration):

```
npm run dev              # both apps
npm run dev:web          # :3000
npm run dev:web:clean    # kills :3000, removes the .next dev lock, then dev:web
npm run dev:workspace    # :3001
npm run build:web
npm run lint | lint:fix | type-check | format
```

`apps/web` also has `lint:css` (stylelint over CSS/SCSS) — a separate check from `lint`.

### Env vars

`src/config/env.config.ts` validates with zod **at import time and throws**, so the app refuses to
boot on a missing variable:

```ts
const configSchema = z.object({
  NEXT_PUBLIC_GA_MEASUREMENT_ID: z.string(),
  NEXT_PUBLIC_SERVER_API_URL: z.string(),
  NEXT_PUBLIC_NOTEBOOK_API_URL: z.string(),
  NEXT_PUBLIC_ASSISTCHAT_API_URL: z.string(),
  NEXT_PUBLIC_APP_URL: z.string(),
  // … 11 in total
});
if (!configProject.success) {
  throw new Error('Các khai báo environment variables không hợp lệ');
}
```

Copy `apps/web/.env.example` → `apps/web/.env` before your first run.

> ### ⚠️ Adding a new `NEXT_PUBLIC_*` variable touches seven more files
> `NEXT_PUBLIC_*` values are **inlined at build time**, not read at runtime. A new one needs:
> 1. the `configSchema` in `env.config.ts`
> 2. its `safeParse` object in the same file (explicit `process.env.X` mapping)
> 3. `apps/web/.env` and 4. `.env.example`
> 5. `turbo.json` → `tasks.build.env` — **miss this and you get stale cached builds**
> 6. `apps/web/Makefile` — **both** `build-com` and `build-vn` `--build-arg` blocks
> 7. `apps/web/Dockerfile` ARG/ENV and 8. `.github/workflows/deploy-develop.yml`
>
> Miss the Makefile or Dockerfile and production ships with the value empty.

---

## 2. The file set

For a list + detail feature, paths relative to `elearning-fe/`:

| File | Role |
|---|---|
| `apps/web/src/app/[locale]/recruit/page.tsx` | route — Server Component, SSR prefetch |
| `apps/web/src/app/[locale]/recruit/layout.tsx` | `<Header/>{children}<Footer/>` + `viewport` |
| `apps/web/src/app/[locale]/recruit/[id]/page.tsx` | detail route |
| `apps/web/src/services/recruit.api.ts` | API service |
| `apps/web/src/modules/recruit/types/index.ts` | types |
| `apps/web/src/modules/recruit/constants/index.ts` | `RECRUIT_PAGE_SIZE = 6` |
| `apps/web/src/shared/queries/useRecruitQueries.tsx` | TanStack Query hooks |
| `apps/web/src/modules/recruit/pages/RecruitPage.tsx` | the list page (client) |
| `apps/web/src/modules/recruit/pages/JobDetailPage.tsx` | the detail page (client) |
| `apps/web/src/modules/recruit/components/*.tsx` | cards, filters, hero, skeleton |
| `apps/web/src/modules/recruit/modals/ApplyCVModal.tsx` | the form |
| `packages/constants/src/route.ts` | `ROUTES.RECRUIT`, `ROUTES.RECRUIT_DETAIL(id)` |
| `packages/constants/src/tanstack-query.ts` | `QUERY_KEYS.RECRUIT.*` |
| `apps/web/messages/*.json` | **9 locale files** |
| `apps/web/src/shared/components/layout/NavigationMenu.tsx` | menu entry |

---

## 3. The HTTP client

`src/core/utils/base-http.ts` (~530 lines) is a `fetch`-based mirror of the CMS's axios client, with
Next-specific extras. Four instances are exported from `src/core/utils/ebook-http.ts`:

```ts
export const ebookHttpPublic = new BaseHttp({ baseURL: `${process.env.NEXT_PUBLIC_SERVER_API_URL}/api/v1/` }).create();
export const ebookHttp       = new BaseHttp({ baseURL: `${…}/api/v1/`, withActionRefresh: true, withActionLogout: true }).create();
export const ebookFeServer    = new BaseHttp({ baseURL: typeof window === 'undefined' ? `${process.env.NEXT_PUBLIC_APP_URL}/` : '/' }).create();
```

| Instance | Use for |
|---|---|
| `ebookHttpPublic` | public backend endpoints |
| `ebookHttp` | authenticated endpoints — handles 401 refresh + logout |
| `ebookFeServer` | the app's **own** `/api/*` route handlers (relative in the browser so LAN-IP access works) |

### It injects three headers automatically

```ts
const accessToken = await getAccessTokenCookie();
if (accessToken) config.headers = { ...config.headers, Authorization: `Bearer ${accessToken}` };
const countryCode = await getCountryCodeCookie();
if (countryCode) config.headers = { ...config.headers, 'w-country': countryCode };
const acceptLanguage = await getNextLocaleCookie();
if (acceptLanguage) config.headers = { ...config.headers, 'accept-language': acceptLanguage };
```

> **`w-country` and `accept-language` drive market-specific pricing and content.** Calling `fetch`
> directly instead of going through `ebookHttp*` silently returns the wrong market's data. This is a
> real, recurring bug.

It also has a dual mode: when a route handler proxies a browser request, pass `forwardRequest` and
the client forwards the incoming headers instead of reading cookies.

### Errors resolve; toasts are automatic

Same contract as the CMS but keyed on a different field:

```ts
const { success, data } = await recruitApi.getJobById(id);
if (!success) return;      // ← the toast already fired
```

The toast sink is `src/lib/handle-http-error.ts`:

```ts
export const handleErrorCode = (errorResponse: ResponseType) => {
  if (isServer) return;                                 // never toast during SSR
  messageInstance.error(errorResponse.message || 'Có lỗi xảy ra');
};
```

`LIST_CODE_ERROR = [422, 400, 403, 402, 409, 404]` — note `404` is included here but not in the CMS.
Pass `offNotify: true` to suppress. **Do not add a second toast.**

Beyond axios parity, `create()` returns `{ request, get, post, put, patch, delete, stream }` and
supports `params` (array-aware), `timeout` (default 60 s, AbortController-based),
`responseType: 'json'|'text'|'blob'|'arrayBuffer'|'stream'`, Next `cache`/`next.revalidate`
passthrough, and `stream()` for SSE.

---

## 4. The service

One file per domain with a module-level `PREFIX`. **Unlike the CMS, the response type is generic** —
this is the better pattern, use it.

```ts
// src/services/recruit.api.ts
import { ebookHttpPublic } from '@/core';
import type { IJob, IJobApplicationPayload, IJobFilters, IJobListParams, IJobListResponse }
  from '@/modules/recruit/types';
import type { ResponseType } from '@/shared/types/http-config.type';

const PREFIX = 'coreservice/job-descriptions';

class RecruitService {
  async getFilters(): Promise<ResponseType<IJobFilters>> {
    return ebookHttpPublic.get(`${PREFIX}/filters`);
  }

  async getJobDescriptions(params: IJobListParams): Promise<ResponseType<IJobListResponse>> {
    return ebookHttpPublic.get(PREFIX, { params });
  }

  async getJobById(id: string): Promise<ResponseType<IJob>> {
    return ebookHttpPublic.get(`${PREFIX}/${id}`);
  }

  async applyJob(payload: IJobApplicationPayload): Promise<ResponseType<void>> {
    return ebookHttpPublic.post('coreservice/job-applications/apply', payload);
  }
}

export const recruitApi = new RecruitService();
```

`ResponseType<T>` is where the generic lands:

```ts
// src/shared/types/http-config.type.ts
export type ResponseType<T = unknown> = Omit<z.infer<typeof BaseResponseSchema>, 'data'> & { data: T };
```

Both a class singleton (`recruit.api.ts`, `auth.api.ts`) and a plain object (`blog.api.ts`) style
exist. Either is fine; match the neighbours. Add the export to `src/services/index.ts`.

## 5. Types

Feature-local types in `src/modules/<f>/types/index.ts`; cross-feature ones in
`packages/types/src/*.types.ts`, imported as `@elearning/types`.

```ts
// src/modules/recruit/types/index.ts
import { Pagination } from '@elearning/types';

export type JobStatus = 'draft' | 'active' | 'expired' | 'closed';
export type EmploymentType = 'all' | 'fulltime' | 'parttime' | 'collaborator' | 'intern' | 'remote';
export enum JobLanguage { Vietnamese = 'vi', English = 'en' }

export interface IJob {
  id: string;
  title: string;
  departmentId: string;
  departmentNameVi: string;
  departmentNameEn: string;
  language: JobLanguage;
  status: JobStatus;
  deadline: string;
  quantity: number;
  salary: string;
  employmentType: EmploymentType;
  content: string;
  createdAt: string;
  updatedAt: string;
  applicationsCount?: number;
}

export interface IJobListResponse {
  job_descriptions: IJob[];      // ⚠ snake_case — the backend leaks it on this endpoint
  pagination: Pagination;
}
```

That `job_descriptions` key is the point: **sibling endpoints return `items`, this one does not.**
Types are hand-written per endpoint; log the payload first.

> **Never import `@elearning/types/src/user.types`** — only the package root. And adding a file to a
> package means updating that package's `src/index.ts` barrel.

## 6. The query hook

Server state goes in TanStack Query. Keys are centralised in
`packages/constants/src/tanstack-query.ts`.

```ts
// src/shared/queries/useRecruitQueries.tsx
export const useRecruitJobs = (params: IJobListParams) => {
  return useQuery<IJobListResponse>({
    queryKey: [QUERY_KEYS.RECRUIT.JOBS, params],
    queryFn: async () => {
      const response = await recruitApi.getJobDescriptions(params);
      return response.data;                 // unwrap the envelope here
    },
    staleTime: 2 * 60 * 1000,
  });
};

export const useRecruitJobDetail = (id: string) => {
  return useQuery<IJob>({
    queryKey: [QUERY_KEYS.RECRUIT.DETAIL, id],
    queryFn: async () => {
      const response = await recruitApi.getJobById(id);
      return response.data;
    },
    staleTime: 5 * 60 * 1000,
    enabled: !!id,                          // don't fire on an empty id
  });
};
```

Cross-feature hooks live in `src/shared/queries/`; feature-local ones in
`src/modules/<f>/hooks/`. Some modules layer a local key factory on top — see
`src/modules/blog/hooks/useBlog.ts`.

Defaults from `src/lib/providers.tsx`: `staleTime` 60 s in production / 10 s in dev,
`refetchOnWindowFocus: false`, `retry: 1`.

**Mutations** go through `src/shared/hooks/useApiMutation.ts`, which unwraps `response.data` and
handles invalidation:

```ts
const mutation = useApiMutation<LoginResponseData, LoginInput, User>({
  apiFn: authApi.login,
  transformResponse: (res) => res.data.user,
  invalidateQueries: (user) => [['users'], ['user', user.id]],
});
```

`invalidateQueries` takes either an array or a function of the response.

## 7. The route

`app/**` files are **Server Components, kept thin** — metadata, prefetch, `headers()`/`cookies()`.
Anything interactive lives in `src/modules/**` behind `'use client'`.

```tsx
// src/app/[locale]/recruit/page.tsx
export default async function Page() {
  const headersList = await headers();
  const isMobile = detectMobileUA(headersList.get('user-agent') ?? '');
  const queryClient = new QueryClient();

  await queryClient.prefetchQuery({
    queryKey: [QUERY_KEYS.RECRUIT.FILTERS],
    queryFn: async () => {
      const response = await recruitApi.getFilters();
      return response.data;
    },
    staleTime: 5 * 60 * 1000,
  });

  return (
    <HydrationBoundary state={dehydrate(queryClient)}>
      <main>
        <RecruitPage isMobile={isMobile} />
      </main>
    </HydrationBoundary>
  );
}
```

The prefetch → `dehydrate` → `<HydrationBoundary>` → same query key in the client component is the
pattern: the client hook finds the data already in cache and renders without a loading flash.

Note `detectMobileUA` on the server passed down as a prop — the codebase prefers this to a
client-only media query for above-the-fold layout.

`layout.tsx` beside it adds chrome:

```tsx
export default function Layout({ children }) {
  return <><Header />{children}<Footer /></>;
}
```

## 8. The list page

```tsx
// src/modules/recruit/pages/RecruitPage.tsx
'use client';
export function RecruitPage({ isMobile }: { isMobile: boolean }) {
  const t = useTranslations('Recruit');
  const { currentPage, setPage } = usePageParam();
  const [searchQuery, setSearchQuery] = useState('');
  const [selectedDept, setSelectedDept] = useState('');

  const { data: filtersData } = useRecruitFilters();

  const jobsParams = useMemo(
    () => ({
      page: currentPage,
      limit: RECRUIT_PAGE_SIZE,
      search: searchQuery || undefined,           // undefined, never ''
      departmentId: selectedDept || undefined,
    }),
    [currentPage, searchQuery, selectedDept]
  );

  const { data: jobsData, isLoading } = useRecruitJobs(jobsParams);
  const jobs = jobsData?.job_descriptions ?? [];
  const total = jobsData?.pagination.total ?? 0;

  const handleDeptChange = useCallback((id: string) => {
    setSelectedDept(id);
    setPage(1);                                   // every filter change resets to page 1
  }, [setPage]);
}
```

Four house rules in that snippet:

- **Pagination lives in the URL.** `usePageParam` does `router.replace` with `scroll: false` and
  deletes `?page` when it is 1 — so the page is shareable and SEO-crawlable.
- **`undefined` means "unset"**, never `''`. Empty strings would be sent and filtered on.
- **`useMemo` the params object**, or the query key changes identity every render and refetches
  forever.
- **Every filter change resets to page 1.**

Rendering: card grids plus `src/shared/components/ui/WPagination.tsx` (an antd `Pagination` that
renders real `<a href>` links and auto-hides when `total <= pageSize`). Loading states are
**dedicated skeleton components** (`JobCardSkeleton`), not spinners. Empty states use `WEmptyData`.

There are no `loading.tsx` or per-route `error.tsx` files — only `src/app/global-error.tsx` and
`src/app/not-found.tsx`.

## 9. The form

**antd `Form` is the default.** react-hook-form appears only in `src/modules/checkout`; do not copy
that pattern elsewhere. `@hookform/resolvers` and zod are installed but **zod is not wired for field
validation anywhere** — it is used only for env validation and one response schema. And
`src/schemas/auth.schema.ts` is entirely commented out, so do not build on it.

```tsx
// src/modules/recruit/modals/ApplyCVModal.tsx
const t = useTranslations('Recruit.applyModal');
const message = useMessage();
const [form] = Form.useForm<FormValues>();

const handleSubmit = async (values: FormValues) => {
  const cvUrl = values.cvFiles?.[0]?.url;
  if (!cvUrl) return;

  setSubmitting(true);
  try {
    const { success } = await recruitApi.applyJob({
      jobDescriptionId: job.id,
      fullName: values.fullName.trim(),
      phoneNumber: values.phoneNumber.trim(),
      email: values.email.trim(),
      cvUrl,
      coverLetter: values.coverLetter?.trim() || undefined,
    });
    if (success) setIsSuccess(true);
    else message.error(t('submitError'));
  } catch {
    message.error(t('submitError'));
  } finally {
    setSubmitting(false);
  }
};
```

Validation rules are inline arrays; shared ones live in `packages/constants/src/rule.ts`:

```ts
export const MAIL_RULES: Rule[] = [
  { required: true, message: 'Email không được để trống' },
  { type: 'email', message: 'Email không hợp lệ' },
];
```

Use `useMessage()` (`src/shared/hooks/useMessage.ts`) in components — it is a theme-aware antd
wrapper with `quickSuccess`/`quickError`. Modals are declarative `open`/`onClose` props on `WModal`;
global ones are registered in `GlobalModal.tsx` and triggered via
`eventBus.$emit('globalAction___openLoginModal')`.

---

## 10. Registering the route

### 10a. `packages/constants/src/route.ts`

```ts
RECRUIT: '/recruit',
RECRUIT_DETAIL: (id: string) => `/recruit/${id}`,
```

Builders for parameterised routes, so links stay typed.

### 10b. Navigation from `@/i18n/routing` — not from `next/*`

```ts
// src/i18n/routing.ts
export const routing = defineRouting({
  locales: AVAILABLE_COUNTRY_CODES,
  defaultLocale: EN_COUNTRY_CODE,
  localeDetection: true,
  localePrefix: 'always',
});
export const { Link, redirect, usePathname, useRouter, getPathname } = createNavigation(routing);
```

> **Import `Link`, `useRouter`, `usePathname` and `redirect` from `@/i18n/routing`**, never from
> `next/link` or `next/navigation`. With `localePrefix: 'always'`, using the Next primitives loses
> the `[locale]` prefix and you get redirected.

### 10c. The menu entry

`src/shared/components/layout/NavigationMenu.tsx` builds `defaultItems` in a `useMemo` from
translation keys plus `getRootCategories()`:

```ts
{ key: 'recruit', label: t('nav.aboutUsDropdown.recruit'), href: ROUTES.RECRUIT }
```

Mobile equivalent: `MobileNavigationDrawer.tsx`. Profile pages use
`src/modules/profile/layout/ProfileSidebarNav.tsx`.

### 10d. Translations — all 9 files

```
apps/web/messages/{vn,en,us,th,ph,my,id,mx,br}.json
```

Top-level namespace is the feature; nest below it:

```jsonc
"Recruit": {
  "hero": { "title": "…" },
  "list": { "resultsFound": "{count} vị trí tuyển dụng" },
  "applyModal": { "submitError": "…" }
}
```

Consume with `useTranslations('Recruit')` (client) or `getTranslations({ locale, namespace })`
(server).

> **A missing key throws in next-intl v4.** It is not a silent fallback like the backend. The drift
> is already real — `en.json`/`vn.json` have 64 top-level keys where the other seven have 63.

### 10e. Auth, if the route needs it

Add the path to `PROTECTED_ROUTES` in `src/proxy.ts`:

```ts
const PROTECTED_ROUTES = [ROUTES.PROFILE];
const AUTH_ROUTES = [ROUTES.LOGIN, ROUTES.REGISTER];
```

`proxy.ts` also handles GeoIP market resolution, the `w_country` cookie, next-intl delegation, https
forcing behind Kong, and `hreflang` rewriting.

> Auth here is **binary** — there are no roles or permissions, unlike the CMS. Login and register are
> **modals**, not pages: `ROUTES.LOGIN` and `ROUTES.REGISTER` exist and `proxy.ts` redirects them,
> but no such page components exist. Trigger the modal via
> `eventBus.$emit('globalAction___openLoginModal')`.

---

## 11. Conventions

| Kind | Convention | Example |
|---|---|---|
| Components | `PascalCase` | `RecruitPage` |
| Props interfaces | `[Name]Props` | `JobCardProps` |
| Types | `[Name]Type`, or `I[Name]` for backend-shaped | `ProductType`, `IJob` |
| Constants | `UPPER_SNAKE_CASE` | `RECRUIT_PAGE_SIZE` |
| Hooks | `useXxx`; mutations `useCreateXxxMutation` | `useRecruitJobs` |
| Service files | `<domain>.api.ts`, export `xxxApi` | `recruit.api.ts` → `recruitApi` |
| Shared components | `W` prefix | `WModal`, `WPagination`, `WUploadFile` |

- **Images**: `next/image`; a new remote host must be added to `next.config.ts` → `images.remotePatterns`.
- **Icons**: `WSvg name="…"` backed by `src/assets/icons/icon-lib.ts`.
- **Theme**: `src/config/theme.config.ts` + `colors.config.ts`; fonts via `next/font` in `font.config.ts`.
- **Barrels**: `src/services/index.ts`, `src/shared/{queries,hooks,store}/index.ts`, and your
  module's `{components,hooks,types,constants,modals}/index.ts`. Not every module has all of them.

---

## 12. Verify

There are **no tests and no test infrastructure** — `turbo.json` has no `test` task, and
`HANDOFF.md` says verification is manual. **Do not promise a test in a PR description.**

```bash
npm run type-check
npm run lint
npm run lint:css
```

Then in the browser:

1. The page renders **server-side** — view source and confirm your content is in the HTML, not just
   hydrated in
2. No loading flash on first paint, if you prefetched correctly
3. Pagination updates the URL; a direct hit on `?page=3` renders page 3
4. Every filter works and resets pagination
5. Switch locale — **all** your strings translate, and none render as a raw key
6. Test at least one non-`vn` locale to catch a missing message file
7. Check the Network tab: requests carry `w-country` and `accept-language`
8. No duplicate error toast on a failed request

---

## Checklist

```
[ ] npm ci run from the repo ROOT
[ ] apps/web/.env created from .env.example
[ ] service in src/services/<domain>.api.ts returning ResponseType<T>
[ ] exported from src/services/index.ts
[ ] types written against the logged payload (watch for snake_case keys)
[ ] query hook with a centralised QUERY_KEYS entry
[ ] key added to packages/constants/src/tanstack-query.ts
[ ] route page.tsx as a thin Server Component with prefetch + HydrationBoundary
[ ] module page as 'use client'
[ ] layout.tsx if the page needs Header/Footer
[ ] ROUTES entry in packages/constants/src/route.ts
[ ] navigation imported from @/i18n/routing, NOT next/link
[ ] NavigationMenu entry (+ mobile drawer)
[ ] translations in ALL 9 messages/*.json
[ ] PROTECTED_ROUTES in src/proxy.ts if authenticated
[ ] skeleton for loading, WEmptyData for empty, error path handled
[ ] no duplicate error toast
[ ] new env var? all 8 files updated
[ ] npm run type-check && npm run lint && npm run lint:css
[ ] manually verified incl. SSR output and a non-vn locale
```
