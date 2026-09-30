# Adding a CRUD screen to the CMS

`elearning-cms` is the internal back-office: React 19 + Vite + Ant Design 6 + Tailwind 4 + Zustand.
Worked through **blog posts** (`src/module/blog/`) — list, create and edit.

What this project deliberately does **not** have, so you do not go looking: no React Query, no
react-hook-form, no zod/yup, no i18n, no tests.

Existing conventions docs: `elearning-cms/.cursor/rules/elearning-cms-frontend.mdc` and
`elearning-cms/UIRULE.md` (UI guardrails — no shadows, gradients, glass effects or emoji).

---

## 1. Getting it running

```bash
cd elearning-cms
npm ci
npm run dev        # → http://localhost:4000
```

> ### ⚠️ Env lives in `env/`, not the repo root
> `vite.config.ts` sets `envDir: './env'`, so Vite reads **`env/.env`** — which is git-ignored, so
> you must create it. The root `elearning-cms/.env` is read **only by the `Makefile`** for Docker
> build args, and says a different port (8002 vs 4000).

```bash
# env/.env
VITE_APP_PORT=4000
VITE_APP_URL=http://localhost:4000
VITE_API_URL=https://staging.api.weupbook.com
VITE_APP_WEB_DOMAIN=https://staging.weupbook.com
```

That is the complete variable list. `VITE_API_URL` is the gateway host; `VITE_APP_WEB_DOMAIN` is used
only for "view on site" links.

```
npm run dev            vite
npm run build          tsc -b && vite build
npm run type-check     tsc --noEmit
npm run lint           eslint .
npm run format         prettier --write "src/**/*.{ts,tsx,js,jsx,json,css,scss,md}"
```

Two things about git hooks: `pre-commit` runs lint-staged, and **`pre-push` runs both `type-check`
and a full `build`**, so pushing is slow and any type error blocks it. Note also that
`tsconfig.app.json` sets `noUnusedLocals` and `noUnusedParameters`, so **an unused import fails the
build** even though ESLint's `no-unused-vars` is off.

---

## 2. The file set

For a list + create + edit feature, paths relative to `elearning-cms/`:

| File | Role |
|---|---|
| `src/services/blogService.ts` | API calls |
| `src/services/index.ts` | barrel — **add your service** |
| `src/module/blog/types/index.ts` | `IBlogPost`, `IBlogPostFilter`, `IBlogPostCreate`, `IBlogPostFormValues` |
| `src/module/blog/constants/index.ts` | status options / labels / colours |
| `src/module/blog/hooks/useBlogPostTable.ts` | data + loading + pagination + `fetchPosts()` |
| `src/module/blog/hooks/useBlogPostColumns.tsx` | antd `ColumnsType` incl. row actions |
| `src/module/blog/hooks/index.ts` | barrel |
| `src/module/blog/pages/BlogPostManager.tsx` | the list page |
| `src/module/blog/pages/CreateBlogPost.tsx` | create page |
| `src/module/blog/pages/UpdateBlogPost.tsx` | edit page |
| `src/module/blog/components/BlogPostForm.tsx` | the shared form, `forwardRef` |
| `src/module/blog/route.tsx` | the module's route table |

Plus **five existing files to edit** — §7.

Shared components you should reuse rather than rebuild:
`src/shared/components/common/ListManagerPageLayout.tsx` (the list shell), `WHeaderForm.tsx` (form
page header), `WImageUpload.tsx`, `WFileUpload.tsx`, `src/shared/components/editor/WEditor.tsx`
(TipTap rich text).

---

## 3. The HTTP client — read this before writing a service

`src/core/baseClient.ts` is the most important file in the project. Three of its behaviours change
how you write every call site.

```ts
// src/core/cmsClient.ts
export const cmsClientPublic: AxiosInstance = new BaseClient({
  baseURL: `${import.meta.env.VITE_API_URL}/api/v1/`,
}).create();

export const cmsClient: AxiosInstance = new BaseClient({
  baseURL: `${import.meta.env.VITE_API_URL}/api/v1/`,
  withActionRefresh: true,
  withActionLogout: true,
}).create();
```

Use `cmsClient` for everything authenticated (almost everything).

### 3.1 Errors never throw

Every response — success or failure — resolves to an `IResponse`:

```ts
// src/shared/types/httpClient.ts
export interface IResponse<T = any> {
  success: boolean;
  error: boolean;
  code: number | string;
  data?: T;
  message?: string;
  rawResponse?: any;
}
```

So the house style is destructure-and-check, never `try/catch`:

```ts
const { data, error } = await blogService.list(params);
if (error) return;          // ← the toast has already been shown for you
```

### 3.2 Error toasts are automatic

The response interceptor fires `message.error(response.data?.message)` for any non-success response
carrying a message:

```ts
// src/shared/helpers/api.ts
export const isNotifyWhenFail = (response: any) => {
  if (response?.data?.success) return false;
  if (response?.config?.offNotify) return false;
  return !!response?.data?.message;
};
```

**You only ever write the success toast.** Adding your own error toast produces a double toast. To
suppress the automatic one — e.g. when you want a custom message on delete — pass `offNotify`:

```ts
return await cmsClient.delete(`coreservice/blogs/${id}`, {
  params: {},
  offNotify: true,
} as ExtendedAxiosRequestConfig);
```

The cast is needed because `offNotify` and `offRefreshToken` are on `ExtendedAxiosRequestConfig`.

### 3.3 `json-bigint` is load-bearing

The request interceptor sets `config.transformResponse = [(data) => data]` to keep the body as a raw
string, then parses it with `JSONbig({ storeAsString: true })`. This exists because backend IDs can
exceed `Number.MAX_SAFE_INTEGER`. **Do not "simplify" it** — you will silently corrupt IDs.

Token refresh on `401` is also handled entirely inside `BaseClient`, including queuing concurrent
failures and replaying them. You never write refresh logic.

---

## 4. The service

One class per domain, exported as a singleton. Endpoints are inline string literals — there is no URL
constants file and no codegen.

```ts
// src/services/blogService.ts
import { cmsClient } from '@/core';
import type { IBlogPostFilter, IBlogPostCreate, IBlogPostUpdate } from '@/module/blog/types';
import type { IResponse, ExtendedAxiosRequestConfig } from '@/shared/types';

class BlogService {
  async list(params: IBlogPostFilter): Promise<IResponse> {
    return await cmsClient.get('coreservice/blogs/list', { params });
  }

  async getById(id: string): Promise<IResponse> {
    return await cmsClient.get(`coreservice/blogs/${id}`);
  }

  async create(payload: IBlogPostCreate): Promise<IResponse> {
    return await cmsClient.post('coreservice/blogs', payload);
  }

  async update(id: string, payload: Partial<IBlogPostUpdate>): Promise<IResponse> {
    return await cmsClient.patch(`coreservice/blogs/${id}`, payload);
  }

  async delete(id: string): Promise<IResponse> {
    return await cmsClient.delete(`coreservice/blogs/${id}`, {
      params: {},
      offNotify: true,
    } as ExtendedAxiosRequestConfig);
  }
}

export const blogService = new BlogService();
```

Note the `coreservice/` prefix on core-service paths; auth paths are bare (`auth/login`).

`IResponse` is used **un-parameterised** here, so `data` is `any`. Type safety stops at the service
boundary and the calling hook re-asserts the shape. That is the house style, not an oversight —
matching it keeps you consistent, though typing new services as `IResponse<T>` is a small
improvement nobody will object to.

Finally, add the export to `src/services/index.ts`.

## 5. Types

```ts
// src/module/blog/types/index.ts
export enum BlogPostStatus {
  ACTIVE = 'active',
  UNPUBLISHED = 'unpublished',
  INACTIVE = 'inactive',
}

export interface IBlogPost {
  id: string;
  slug: string;
  title: string;
  description: string;
  blogContent: string;
  market: MarketType;
  banner: string;
  thumbnail: string;
  status: BlogPostStatus;
  publishDate: string;
  categories: IBlogCategory[];
  createdAt: string;
  updatedAt: string;
}

export interface IBlogPostFilter {
  offset: number;
  limit: number;
  title?: string;
  market?: string;
  categoryId?: string;
  sortBy?: string;
  sortOrder?: string;
}

export interface IBlogPostCreate { /* the API payload — ISO date string, no categories */ }
export interface IBlogPostUpdate extends IBlogPostCreate {}

export interface IBlogPostFormValues {
  /* … same fields, plus: */
  categoryIds: string[];
  publishDate: any;        // dayjs object, not a string
}
```

**The split between `I…FormValues` and `I…Create` is deliberate.** Form values hold a dayjs object
and the relation IDs; the API payload holds an ISO string and no relations. Keeping them separate is
what lets the submit handler do the conversion in one obvious place.

Conventions: `I` prefix for interfaces, `PascalCase` enums with `UPPER_SNAKE` members. Types are
hand-written to mirror backend DTOs — **there is no codegen, so log the real payload once before
writing them.**

## 6. The data hook

Every list feature copies this shape. It is the project's substitute for React Query.

```ts
// src/module/blog/hooks/useBlogPostTable.ts
export const useBlogPostTable = () => {
  const [data, setData] = useState<IBlogPost[]>([]);
  const [loading, setLoading] = useState(false);
  const [pagination, setPagination] = useState({ page: 1, limit: 20, total: 0, totalPages: 0 });
  const [lastFetchTime, setLastFetchTime] = useState<Date>(new Date());

  const fetchPosts = useCallback(async (filters: IBlogPostFilter) => {
    setLoading(true);
    try {
      const params: IBlogPostFilter = { offset: filters.offset, limit: filters.limit };
      if (filters.title) params.title = filters.title;        // ← omit empty keys, never send undefined
      if (filters.market) params.market = filters.market;
      if (filters.categoryId) params.categoryId = filters.categoryId;

      const { data: responseData, error } = await blogService.list(params);
      if (error) {
        setData([]);
        return;                                               // ← toast already fired
      }

      setData(responseData?.items || []);

      if (responseData?.pagination) {
        const { page, offset, limit, total, totalPages } = responseData.pagination;
        const currentPage = page ?? (offset !== undefined ? Math.floor(offset / limit) + 1 : 1);
        setPagination({ page: currentPage, limit, total, totalPages });
      }
    } catch (err) {
      console.error(err);
      setData([]);
    } finally {
      setLastFetchTime(new Date());
      setLoading(false);
    }
  }, []);

  return { data, loading, lastFetchTime, pagination, fetchPosts, setPagination };
};
```

Two details worth copying verbatim:

- **Conditional param assignment.** The backend runs `forbidNonWhitelisted: true`, and sending
  `title: undefined` or `title: ''` either 400s or filters on an empty string. Build the object up.
- **Pagination normalisation.** The `page ?? (offset …)` line exists because endpoints disagree about
  `page` vs `offset`. Blog posts use `offset`; blog categories use `page`.

Columns live in a separate `use<X>Columns.tsx` (`.tsx` because it returns JSX) taking `onEdit` /
`onDelete` callbacks.

## 7. Registering the route — five places

**Skipping any of these fails quietly.** This is the CMS equivalent of core-service's silent
registrations.

### 7a. `src/shared/constants/route.ts`

```ts
BLOG_POSTS: '/blog/posts',
BLOG_CREATE_POST: '/blog/posts/create',
BLOG_EDIT_POST: '/blog/posts/edit/:blog-id',
```

The table is `Object.freeze({...})`.

### 7b. The module's `route.tsx`

```tsx
// src/module/blog/route.tsx
import { type RouteObject } from 'react-router-dom';
import { lazy } from 'react';
import { ROUTE_NAME } from '@/shared/constants';

const BlogPostManager = lazy(() => import('./pages/BlogPostManager'));
const CreateBlogPost = lazy(() => import('./pages/CreateBlogPost'));
const UpdateBlogPost = lazy(() => import('./pages/UpdateBlogPost'));

export const blogPostRoutes: RouteObject[] = [
  { path: ROUTE_NAME.BLOG_POSTS, element: <BlogPostManager /> },
  { path: ROUTE_NAME.BLOG_CREATE_POST, element: <CreateBlogPost /> },
  { path: ROUTE_NAME.BLOG_EDIT_POST, element: <UpdateBlogPost /> },
];

export default blogPostRoutes;
```

Use `lazy()`. (`src/module/career/route.tsx` does not, and is the outlier.)

### 7c. `src/router/routes.tsx`

Spread it into the guarded children array:

```tsx
{
  loader: authGuard,
  element: <AuthRequiredLayout><Outlet /></AuthRequiredLayout>,
  children: [
    { path: ROUTE_NAME.HOME, element: <Home /> },
    ...productRoutes,
    ...blogPostRoutes,
  ],
}
```

### 7d. `src/shared/helpers/permission.ts` — `ROUTE_PERMISSION_MAP`

```ts
export const ROUTE_PERMISSION_MAP: Record<string, PermissionGroup | WildcardPermission> = {
  [ROUTE_NAME.BLOG_POSTS]: PERMISSIONS.BLOG_POSTS,
  [ROUTE_NAME.BLOG_CREATE_POST]: PERMISSIONS.BLOG_POSTS,
  [ROUTE_NAME.BLOG_EDIT_POST]: PERMISSIONS.BLOG_POSTS,
};
```

> **Skip this and the route behaves confusingly.** `canAccessRoute` falls back to
> `if (!permissionGroup) return pathname === '/'`, so an unmapped non-root path returns **false** and
> the user gets a `403` page they cannot explain.

### 7e. `src/layout/components/SideBar.tsx` — `AllMenuItems`

A single ~210-line array of `MenuItem`:

```ts
interface MenuItem {
  name: string;
  routeName: string;
  icon: React.ReactNode;
  requiredPermission?: MenuPermission;
  requireReadPermission?: boolean;
  children?: { /* … */ }[];
}
```

Filtered by permission in a `useMemo`; a parent with no visible children is hidden entirely.
**Skip this and the page exists but is unreachable by clicking.**

If your feature needs a permission group that does not exist yet, also add it to
`src/shared/constants/permissions.ts`.

> Note: `PERMISSIONS.BLOG_POSTS` and `PERMISSIONS.BLOG_CATEGORIES` are **both** `'blog'`. That is
> intentional — one backend resource — and `getAllowedPermissions()` de-dupes for it.
>
> Also note the permission constants are importable from **two** paths:
> `@/shared/constants` and `@/shared/types/role`. `permission.ts` uses the latter, `SideBar.tsx` the
> former. Match the file you are editing.

---

## 8. The list page

```tsx
// src/module/blog/pages/BlogPostManager.tsx
const columns = useBlogPostColumns({ onEdit: handleEdit, onDelete: handleDelete });

return (
  <ListManagerPageLayout
    title="Bài đăng"
    columns={columns}
    dataSource={data}
    loading={loading}
    pagination={pagination}
    onPageChange={handlePageChange}
    onPageSizeChange={handlePageSizeChange}
    lastFetchTime={lastFetchTime}
    onReload={() => fetchWithFilters()}
    filterContent={<div className="flex w-full gap-6 lg:w-2/3 lg:flex-row">{/* Input + Selects */}</div>}
    actionContent={
      <Button type="primary" icon={<PlusOutlined />} className="text-white! shadow-none"
              onClick={handleCreate} size="large">Tạo bài đăng</Button>
    }
  />
);
```

`ListManagerPageLayout` gives you the title with count, filter row, action row, antd `Table`, and a
footer with pagination, a page-size `Select` and a "last updated" reload control.

Delete is an antd `Modal.confirm`, with page-back-off when the last row on a page is removed:

```tsx
Modal.confirm({
  title: 'Xoá bài viết blog',
  icon: <ExclamationCircleOutlined />,
  content: 'Bạn có chắc chắn muốn xoá bài viết blog này?',
  okText: 'Xoá', okType: 'danger', cancelText: 'Huỷ',
  onOk: async () => {
    const { error } = await blogService.delete(id);
    if (error) return;
    message.success('Xoá bài viết thành công');
    const nextPage = size(data) === 1 && pagination.page > 1 ? pagination.page - 1 : pagination.page;
    setPagination((prev) => ({ ...prev, page: nextPage }));
    fetchWithFilters(nextPage, pagination.limit);
  },
});
```

## 9. The form

One `forwardRef` form shared between create and edit, so the **page** owns the submit button:

```tsx
// src/module/blog/components/BlogPostForm.tsx
const BlogPostForm = forwardRef(({ initialValues, onSubmit }: BlogPostFormProps, ref) => {
  const [form] = Form.useForm<IBlogPostFormValues>();

  useImperativeHandle(ref, () => ({
    submit: () => form.submit(),
    getValues: () => form.getFieldsValue(),
  }));

  useEffect(() => {
    if (initialValues) form.setFieldsValue(initialValues);
  }, [initialValues, form]);

  return (
    <Form form={form} layout="vertical"
          initialValues={{ status: BlogPostStatus.ACTIVE, ...initialValues }}
          onFinish={onSubmit} className="w-full">
      <Form.Item label={<span className="font-semibold">Thị trường</span>} name="market"
                 rules={[{ required: true, message: 'Vui lòng chọn thị trường' }]}>
        <Select placeholder="Chọn thị trường" size="large" options={marketOptions} />
      </Form.Item>

      <Form.Item name="blogContent"
        rules={[{ validator: (_, value) =>
          value && value.trim()
            ? Promise.resolve()
            : Promise.reject(new Error('Nội dung bài viết không được để trống')) }]}>
        <WEditor minHeight="calc(100vh - 350px)" />
      </Form.Item>
    </Form>
  );
});
BlogPostForm.displayName = 'BlogPostForm';
```

**Validation is antd `rules` arrays, inline.** There is no schema library. Use `{ required: true,
message }` for simple cases and a `validator` function returning a promise for anything else.

The create page wires it up:

```tsx
const formRef = useRef<any>(null);

const handleSubmit = async (values: IBlogPostFormValues) => {
  const payload = {
    /* … */
    publishDate: values.publishDate
      ? dayjs(values.publishDate).toISOString()
      : new Date().toISOString(),
  };
  const { data: createData, error } = await blogService.create(payload);
  const created = createData?.data?.blog;          // ⚠ double-nested on create — see below
  if (error || !created?.id) return;
  if (values.categoryIds?.length > 0) {
    await blogService.attachCategories(created.id, values.categoryIds);
  }
  message.success('Tạo bài viết thành công');
  navigate(ROUTE_NAME.BLOG_POSTS);
};

// …
<WHeaderForm title="Tạo bài đăng" />
<BlogPostForm ref={formRef} onSubmit={handleSubmit} />
<Button type="primary" size="large" loading={loading} onClick={() => formRef.current?.submit()}>Tạo</Button>
```

> ### ⚠️ Response nesting is not uniform
> On blog posts specifically:
> - `list` → `data.items` + `data.pagination`
> - `getById` → `data.blog ?? data` (the edit page defends against both)
> - `create` → **`data.data.blog`**
>
> That last one is the backend interceptor's `data.data` nesting — see
> [`02-api-contract.md`](02-api-contract.md#the-datadata-trap). **Log the payload before writing the
> access path.**

The edit page additionally diffs relations rather than replacing them:

```ts
const toAttach = newCategoryIds.filter((c) => !originalCategoryIds.includes(c));
const toDetach = originalCategoryIds.filter((c) => !newCategoryIds.includes(c));
await Promise.all([
  toAttach.length ? blogService.attachCategories(id, toAttach) : Promise.resolve(),
  toDetach.length ? blogService.detachCategories(id, toDetach) : Promise.resolve(),
]);
```

For a small entity, a modal is the lighter alternative — see
`src/module/blogCategory/modals/CreateUpdateBlogCategoryModal.tsx`. Modals here are imperative:
`useImperativeHandle(ref, () => ({ open: (data) => … }))` plus `onClose`/`onSuccess` callbacks.

---

## 10. Conventions

| Kind | Convention |
|---|---|
| Pages / components | `PascalCase.tsx` |
| Hooks | `useXxx.ts`, or `.tsx` when they return JSX |
| Services | `xxxService.ts`, exported as a singleton instance |
| Stores | `xxxStore.ts` (zustand) |
| Shared components | `W` prefix — `WEditor`, `WImageUpload`, `WHeaderForm` |
| Imports | always `@/…`, never deep relatives |
| Constants | `SCREAMING_SNAKE_CASE` |
| Dates | `dayjs` only, via `src/shared/helpers/time.ts` |

- **No i18n.** Write Vietnamese strings directly, matching neighbours.
- **Tailwind 4 + antd specificity** — use the `class!` important suffix as the codebase does
  (`text-white!`, `m-0!`). Do not reach for `styled` or inline styles. The `@layer` ordering in
  `src/assets/styles/index.css` is what makes this necessary.
- **Theme** is `src/shared/constants/theme.ts` — `colorPrimary: '#013d91'`, `borderRadius: 8`, font
  Readex Pro.
- **Barrels to update**: `src/services/index.ts`, and your module's `hooks/`, `components/`,
  `modals/` and `types/` index files. Not every module has all of them — check first.
- Module folder naming is inconsistent (`blogCategory` camel, `access-control` kebab). Match the
  neighbours.

---

## 11. Verify

There are **no tests and no test infrastructure**. Verification is:

```bash
npm run type-check
npm run lint
```

Then manually, in the browser:

1. List loads; pagination, page-size change and reload all work
2. Each filter narrows results, and clearing it restores them
3. Create → success toast → redirect → the new row appears
4. Edit → values pre-populate → save → changes persist after reload
5. Delete → confirm dialog → row disappears; deleting the last row on page 2 drops you to page 1
6. **Log in as a non-admin without the permission** — the sidebar entry must be hidden and direct
   URL navigation must show the 403 page
7. Check the browser console for a double toast on a failed request (means you added one the
   interceptor already fired)

---

## Checklist

```
[ ] env/.env created (not the root .env)
[ ] service class in src/services/ + exported from src/services/index.ts
[ ] types written against the LOGGED payload, not assumed
[ ] I…FormValues kept separate from I…Create
[ ] use<X>Table hook with conditional param assignment + pagination normalisation
[ ] use<X>Columns hook
[ ] list page using ListManagerPageLayout
[ ] form as a forwardRef component shared by create and edit
[ ] validation as antd rules arrays
[ ] ROUTE_NAME entries
[ ] module route.tsx with lazy()
[ ] spread into src/router/routes.tsx
[ ] ROUTE_PERMISSION_MAP entries          ← silent failure if skipped
[ ] AllMenuItems entry in SideBar.tsx     ← silent failure if skipped
[ ] PERMISSIONS constant if new
[ ] no try/catch around HTTP calls — checked `error` instead
[ ] no duplicate error toast
[ ] barrels updated
[ ] npm run type-check && npm run lint
[ ] manually verified, including as a user without the permission
```
