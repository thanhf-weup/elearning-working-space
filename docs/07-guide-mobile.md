# Adding a feature to the mobile app

`elearning-mobile-app` is Flutter 3.41 + **Riverpod 3 as both state management and DI**, feature-first
MVVM. Worked through **blog** (`lib/features/blog/`) — a list screen with filter, search and infinite
scroll, plus a detail screen.

Deliberately not used, so stop looking for them: bloc, provider, get_it/injectable, retrofit,
auto_route, hive/isar/sqflite, easy_localization, melos, build flavours. **There are also no use-case
files** — controllers call repositories directly, which `ARCHITECTURE.md` sanctions.

This app has the most thorough existing documentation of the five. Read alongside:
`elearning-mobile-app/ARCHITECTURE.md`, `guideline-core-convention/00..08-*.md` (the actual
rulebook), and `lib/features/_feature_template/` (a copy-me skeleton).

---

## 1. Getting it running

```bash
cd elearning-mobile-app
make setup                 # flutter pub get + make gen
make run ENV=dev
```

> ### ⚠️ Two things will waste your first hour
> **1. A fresh clone does not compile.** `*.freezed.dart`, `*.g.dart` and
> `lib/l10n/app_localizations*.dart` are all **gitignored**. Run `make setup` first, and `make gen`
> after every pull and after every change to an ARB file, a Freezed model or an annotation.
>
> **2. Never run `flutter run` or `flutter build` bare.** Without `--dart-define-from-file` the app
> falls back to `Env` defaults (`http://localhost:4000/api/v1`) and silently talks to nothing. Always
> `make run ENV=…`.

Environments are **compile-time only** via `--dart-define-from-file`. There are no Android/iOS
flavours.

| File | Env | Committed |
|---|---|---|
| `config/dev.json` | `ENV=dev` | yes |
| `config/staging.json` | `ENV=staging` | yes |
| `config/prod.json` | `ENV=prod` | yes, but **intentionally broken** |
| `config/dev.local.json` | local host override, layered **after** dev.json | no (gitignored) |

`config/prod.json` holds `https://api.replace-me.invalid/api/v1`. `make check-env` refuses
`ENV=prod` while that placeholder is present, and `Env.validate()` throws at startup. That is
deliberate, not a bug.

For local backend work, copy `config/dev.local.example.json` → `config/dev.local.json`:

| Target | Host |
|---|---|
| Android emulator | `http://10.0.2.2:4000/api/v1` — **emulators cannot reach `localhost`** |
| iOS simulator | `http://localhost:4000/api/v1` |
| Physical device | `http://192.168.1.x:4000/api/v1` |

### The Makefile is the only interface

| Command | What it does |
|---|---|
| `make setup` | `flutter pub get` + `make gen` |
| `make gen` | `check-l10n` → `flutter gen-l10n` → `build_runner build` |
| `make gen-watch` | l10n once, then `build_runner watch` |
| `make check-l10n` | `dart run tool/check_l10n.dart` |
| `make format` / `format-check` | `dart format lib test tool` |
| `make analyze` | `gen` then `flutter analyze` |
| `make test` / `test-coverage` | `gen` then `flutter test` (+ coverage gate) |
| **`make check`** | **gen + format-check + analyze + test — the pre-commit gate** |
| `make run [ENV=…]` | `flutter run` with the right dart-defines |
| `make run-profile` / `run-release` | AOT — needed for iOS physical devices |
| `make build-apk ENV=prod` | release APK, obfuscated, split debug info |
| `make clean` | `flutter clean` + deletes all generated `.dart` files |

`build-*` targets depend on `check`, so they run the full gate first. `.vscode/launch.json` has four
ready configs (dev, dev local, staging, production).

**This is the only repo with a CI gate that can fail on code quality** — `.github/workflows/ci.yml`
runs format, analyze, test and a per-file coverage floor (`tool/check_coverage.dart`) on PRs to
`main`. The floor covers critical infrastructure only: `result.dart` 90%, `jwt_utils.dart` 90%,
`auth_interceptor.dart` 85%, `auth_token_store.dart` 85%, and so on.

Lint (`analysis_options.yaml`): `flutter_lints` plus `always_use_package_imports`,
`avoid_dynamic_calls`, `avoid_print`, `directives_ordering`, `prefer_final_locals`,
`use_super_parameters`.

---

## 2. The layer chain

```
presentation  ──►  domain  ◄──  data  ──►  core
```

Build in dependency order — domain first, presentation last:

```
features/blog/
├── blog_providers.dart                    ← DI composition root
├── data/
│   ├── datasources/blog_remote_data_source.dart    ← the HTTP call + envelope validation
│   ├── models/blog_item_model.dart                 ← Freezed DTO
│   ├── mappers/blog_mapper.dart                    ← DTO → entity
│   └── repositories/blog_repository_impl.dart      ← error mapping to Result
├── domain/
│   ├── entities/blog.dart                          ← plain Dart
│   └── repositories/blog_repository.dart           ← abstract interface class
└── presentation/
    ├── controllers/blog_list_controller.dart       ← AsyncNotifier + state
    ├── models/blog_format.dart                     ← presentation-only helpers
    ├── screens/blog_list_screen.dart
    └── widgets/blog_*.dart
```

Dependency rules (DEP-1..8) that the reviewer will check:

- `domain` imports no Flutter, Riverpod, Dio, `data` or `presentation`
- `data` never imports `presentation`
- **`presentation` imports the repository contract and the provider, never the impl**
- `core` never imports `features`, and reads only `app/config/env.dart` from `app`
- a feature never imports another feature's `data` or `presentation`
- only `app/router` imports feature pages

**No barrel files exist.** Every import is an explicit `package:weup_elearning/...` path —
`always_use_package_imports` makes relative imports a lint error.

---

## 3. Register the endpoint path

**Never inline a URL in a datasource** (rule ARCH-5). All paths live in one registry:

```dart
// lib/core/network/api_endpoints.dart
static const blogList = '/coreservice/blogs/list';
static const blogBySlug = '/coreservice/blogs/slug';
static const blogCategories = '/coreservice/blog-categories/list';
```

The datasource aliases them as statics (DATA-9).

## 4. The datasource

```dart
// lib/features/blog/data/datasources/blog_remote_data_source.dart
/// Ba endpoint blog đều public nên gọi với [NetworkAuthMode.none].
final class BlogRemoteDataSource {
  const BlogRemoteDataSource(this._apiClient);

  static const listEndpoint = ApiEndpoints.blogList;
  static const detailEndpoint = ApiEndpoints.blogBySlug;

  final ApiClient _apiClient;

  Future<BlogListPageModel> getBlogs({
    int offset = 0,
    int limit = 11,
    String? categoryId,
    String? title,
  }) async {
    if (offset < 0) {
      throw ArgumentError.value(offset, 'offset', 'offset phải lớn hơn hoặc bằng 0.');
    }

    final trimmedTitle = title?.trim();

    final response = await _apiClient.requestJson<Map<String, dynamic>>(
      listEndpoint,
      queryParameters: <String, dynamic>{
        'offset': offset,
        'limit': limit,
        'status': 'active',
        if (trimmedTitle != null && trimmedTitle.isNotEmpty) 'title': trimmedTitle,
      },
      authMode: NetworkAuthMode.none,
    );

    final data = _unwrapData(response.data, 'blogs/list');

    final items = data['items'];
    if (items is! List) {
      throw const FormatException('blogs/list response thiếu items[].');
    }

    try {
      return BlogListPageModel.fromJson(Map<String, dynamic>.from(data));
    } on TypeError catch (error) {
      throw FormatException('blogs/list item sai kiểu: $error');
    }
  }

  /// Bóc `data` khỏi envelope chung, ném [FormatException] nếu shape sai.
  Map<Object?, Object?> _unwrapData(Map<String, dynamic>? envelope, String endpointLabel) {
    if (envelope == null || envelope['success'] != true) {
      throw FormatException('$endpointLabel response thiếu success=true.');
    }
    final data = envelope['data'];
    if (data is! Map) {
      throw FormatException('$endpointLabel response thiếu object data.');
    }
    return data;
  }
}
```

Five things to copy exactly:

1. **Validate the envelope before `fromJson`** (DATA-11). `_unwrapData` is the canonical helper —
   copy it into your datasource.
2. **Conditional query params.** Omit empty values rather than sending blank strings, and remember
   the backend 400s on undeclared keys.
3. **Wrap `TypeError` into `FormatException`.** Codegen throws `TypeError` on a shape mismatch, and
   the repository's `_guard` maps `FormatException` to a clean `DataFailure`.
4. **Argument validation up front** with `ArgumentError.value` for programmer errors.
5. **`authMode: NetworkAuthMode.none` for public endpoints.**

> ### ⚠️ `NetworkAuthMode` defaults to `session` — fail-closed
> ```dart
> enum NetworkAuthMode { session, none }   // default is session
> ```
> A public endpoint called without `authMode: NetworkAuthMode.none` gets a stale bearer token
> attached. Conversely, a new **public `/auth/*` endpoint** must also be added to
> `AuthInterceptor._publicAuthPaths`, or the interceptor will try to refresh against it.

### Envelope shapes differ per endpoint

Documented per feature in `lib/features/<f>/api.md` — write one for your feature too:

| Endpoint | Data path | Pagination |
|---|---|---|
| `blogs/list` | `data.items[]` + `data.pagination` | `offset` |
| `blogs/slug/{slug}` | **`data.blog`** — one level deeper | — |
| `blog-categories/list` | `data.items[]` | **`page`** |
| `auth/login` | `data.token` | — |

## 5. The model — Freezed + json_serializable

```dart
// lib/features/blog/data/models/blog_item_model.dart
import 'package:freezed_annotation/freezed_annotation.dart';

part 'blog_item_model.freezed.dart';
part 'blog_item_model.g.dart';

@freezed
abstract class BlogItemModel with _$BlogItemModel {
  const factory BlogItemModel({
    required String id,
    required String title,
    required String slug,
    @Default('') String description,
    @Default('') String blogContent,
    String? banner,
    String? thumbnail,
    @Default('active') String status,
    @Default('vn') String market,
    String? publishDate,                 // kept as String; the mapper parses it
    @Default(<BlogCategoryModel>[]) List<BlogCategoryModel> categories,
  }) = _BlogItemModel;

  factory BlogItemModel.fromJson(Map<String, dynamic> json) => _$BlogItemModelFromJson(json);
}
```

- **Freezed 3 syntax** is `abstract class X with _$X`, not `class X with _$X`.
- **Every field is nullable or defaulted**, so a missing or extra backend field never crashes.
- **Dates stay `String` in the data layer.** Parsing is the mapper's job.
- `@JsonKey(name: 'display_name')` for snake_case renames.
- Then `make gen`.

## 6. The mapper

An extension on the model. This is where the data layer's leniency becomes the domain's strictness:

```dart
// lib/features/blog/data/mappers/blog_mapper.dart
extension BlogItemModelMapper on BlogItemModel {
  Blog toEntity() {
    final blogId = id.trim();
    if (blogId.isEmpty) {
      throw const FormatException('Blog thiếu id.');
    }

    final blogSlug = slug.trim();
    if (blogSlug.isEmpty) {
      // Không có slug thì card không điều hướng được sang màn chi tiết.
      throw FormatException('Blog $blogId thiếu slug.');
    }

    return Blog(
      id: blogId,
      title: title.trim(),
      slug: blogSlug,
      description: description.trim(),
      blogContent: blogContent,
      banner: _nullIfEmpty(banner),
      thumbnail: _nullIfEmpty(thumbnail),
      publishDate: _parsePublishDate(blogId),
      categories: categories.toEntities(),
    );
  }
}
```

Throw `FormatException` for anything the UI genuinely cannot render, and fall back for anything it
can — `_parsePublishDate` tries `publishDate`, then `createdAt`, before giving up.

## 7. The entity

Plain Dart. No Flutter, no Dio, no Riverpod, no JSON.

```dart
// lib/features/blog/domain/entities/blog.dart
/// Bài blog mà presentation hiểu; không phụ thuộc JSON hay Dio.
final class Blog {
  Blog({
    required this.id,
    required this.title,
    required this.slug,
    required this.publishDate,
    required List<BlogCategory> categories,
    this.banner,
    this.thumbnail,
  }) : categories = List.unmodifiable(categories);

  final String id;
  final String title;
  final String slug;
  final DateTime publishDate;
  final List<BlogCategory> categories;
  final String? banner;
}
```

`List.unmodifiable` in the initialiser is the house pattern for collection fields. Entity file names
are the plain business noun with **no suffix**.

## 8. The repository — contract and impl

The contract lives in `domain/`, returns `Result<T>`, and never leaks Dio:

```dart
// lib/features/blog/domain/repositories/blog_repository.dart
abstract interface class BlogRepository {
  Future<Result<BlogPage>> getBlogs({int offset, int limit, String? categoryId, String? title});
  Future<Result<Blog>> getBlogDetail({required String slug});
  Future<Result<List<BlogCategory>>> getCategories({int page, int limit});
}
```

The impl's whole job is mapping raw errors into `Failure`:

```dart
// lib/features/blog/data/repositories/blog_repository_impl.dart
final class BlogRepositoryImpl implements BlogRepository {
  const BlogRepositoryImpl(this._remoteDataSource);
  final BlogRemoteDataSource _remoteDataSource;

  @override
  Future<Result<Blog>> getBlogDetail({required String slug}) {
    return _guard(() async {
      final blog = await _remoteDataSource.getBlogDetail(slug: slug);
      return blog.toEntity();
    });
  }

  /// Đổi mọi lỗi thô của datasource thành [Failure] trước khi trả về domain.
  Future<Result<T>> _guard<T>(Future<T> Function() run) async {
    try {
      return Ok<T>(await run());
    } on DioException catch (error) {
      return Err<T>(mapDioException(error));
    } on FormatException {
      return Err<T>(const DataFailure());
    } on TypeError {
      return Err<T>(const DataFailure());
    } on Object {
      return Err<T>(const UnknownFailure());
    }
  }
}
```

Copy `_guard` verbatim. `mapDioException` (`lib/core/network/api_failure_mapper.dart`) already
handles all three backend envelope variants and sanitises suspicious messages — a backend `message`
over 200 characters is discarded as a likely SQL or stack trace, falling back to a friendly
per-status default.

`Failure` (`lib/core/errors/failure.dart`) is a sealed hierarchy carrying a **stable client `code`,
never the HTTP status**, a display-safe `message`, `isRetryable` and `isSilent`. Subclasses include
`NetworkFailure`, `TimeoutFailure`, `UnauthorizedFailure`, `ValidationFailure`,
`RateLimitedFailure`, `ServerFailure`, `DataFailure`, `UnknownFailure`.

## 9. DI — the feature's composition root

One file per feature, and **the only place a datasource or repository is constructed** (STATE-3):

```dart
// lib/features/blog/blog_providers.dart
/// Composition root của feature blog: nơi duy nhất `new` datasource/repository.

final blogRemoteDataSourceProvider = Provider<BlogRemoteDataSource>((ref) {
  return BlogRemoteDataSource(ref.watch(apiClientProvider));
});

/// Khai báo theo interface `BlogRepository` để controller/test override được.
final blogRepositoryProvider = Provider<BlogRepository>((ref) {
  return BlogRepositoryImpl(ref.watch(blogRemoteDataSourceProvider));
});
```

**Type the provider as the interface**, not the impl — that is what lets tests override it.

Never construct a `Dio`; `apiClientProvider` comes from `lib/core/network/network_providers.dart`,
which wires three instances: `dioProvider` (core + auth, with `AuthInterceptor` and
`MarketInterceptor`), `notebookDioProvider` (Studio AI, different host, sharing the same
`AuthRefreshGate`), and `refreshDioProvider` (deliberately **without** `AuthInterceptor` to avoid
refresh recursion).

Controller providers live in their own controller file, not here.

## 10. The controller

```dart
// lib/features/blog/presentation/controllers/blog_list_controller.dart
final blogListControllerProvider =
    AsyncNotifierProvider.autoDispose<BlogListController, BlogListState>(
      BlogListController.new,
      // Tắt auto-retry mặc định của Riverpod 3: màn này có nút "Thử lại" tường minh.
      retry: (retryCount, error) => null,
    );

final class BlogListController extends AsyncNotifier<BlogListState> {
  static const pageSize = 11;
  static const searchDebounce = Duration(milliseconds: 400);

  String? _categoryId;
  String _search = '';
  int _requestVersion = 0;         // race guard — the house pattern
  Timer? _debounce;

  @override
  Future<BlogListState> build() {
    ref.onDispose(() {
      _requestVersion++;
      _debounce?.cancel();
    });
    return _fetchFirstPage();
  }

  Future<BlogListState> _fetchFirstPage() async {
    final result = await ref.read(blogRepositoryProvider).getBlogs(
      limit: pageSize,
      categoryId: _categoryId,
      title: _search.isEmpty ? null : _search,
    );
    return result.fold(onSuccess: _toState, onFailure: (failure) => throw failure);
  }

  Future<void> _reload({bool showLoading = true}) async {
    final requestVersion = ++_requestVersion;
    if (showLoading) state = const AsyncLoading<BlogListState>();
    final nextState = await AsyncValue.guard(_fetchFirstPage);
    if (requestVersion == _requestVersion) state = nextState;   // drop a stale response
  }
}
```

Four Riverpod 3 specifics:

- **`retry: (retryCount, error) => null`** whenever the screen has an explicit retry button.
  Riverpod 3 retries with backoff by default, which fires invisible background requests and makes
  the error state flicker.
- **`_requestVersion` race guard.** Increment before the request, compare after; discard if it moved.
  Without it a slow first request overwrites a fast second one.
- **`throw failure` in `onFailure`**, so the `Failure` lands in `AsyncError` and the screen can read
  `error is Failure ? error.message : …`.
- **`.family` arguments arrive via the constructor**, not `build(arg)`:
  ```dart
  final blogDetailControllerProvider = AsyncNotifierProvider.autoDispose
      .family<BlogDetailController, Blog, String>(BlogDetailController.new, retry: (c, e) => null);

  final class BlogDetailController extends AsyncNotifier<Blog> {
    BlogDetailController(this.slug);
    final String slug;
  ```

Choosing the right provider (`05-state-management-rules.md`): read-only dependency → `Provider`;
fetch plus actions → `AsyncNotifier`; simple GET with no actions → `FutureProvider`; multi-action
sync state → `NotifierProvider`; a stream → `StreamProvider`; purely local UI → `StatefulWidget`.

## 11. The screen — four states, always

```dart
// lib/features/blog/presentation/screens/blog_list_screen.dart
final l10n = AppLocalizations.of(context);
final state = ref.watch(blogListControllerProvider);
final controller = ref.read(blogListControllerProvider.notifier);

return Scaffold(
  backgroundColor: AppColors.background,
  body: Column(children: [
    BlogListHeader(controller: _searchController, onSearchChanged: controller.setSearch),
    Expanded(child: RefreshIndicator(
      onRefresh: controller.refresh,
      child: CustomScrollView(
        controller: _scrollController,
        physics: const AlwaysScrollableScrollPhysics(),
        slivers: [
          ...state.when(
            loading: () => _loadingSlivers(context),
            error: (error, stackTrace) => [
              SliverFillRemaining(
                hasScrollBody: false,
                child: BlogErrorView(
                  message: error is Failure ? error.message : l10n.blogLoadError,
                  onRetry: controller.retry,
                ),
              ),
            ],
            data: (data) => _blogSlivers(context: context, state: data, onBlogTap: _openDetail),
          ),
        ],
      ),
    )),
  ]),
);
```

Rules UI-10..13 require **four** states on every API-backed screen: loading, data, **empty**, and
error-with-retry. Shared widgets: `lib/shared/widgets/state_views/{error_state_view,empty_state_view}.dart`
and `app_snack_bar.dart` (`AppSnackBar.showSuccess/showError/showWarning`). A feature may keep local
variants when the layout differs — `blog_state_views.dart` does.

Navigation is always by **name**, never a route literal (NAME-7, UI-14):

```dart
void _openDetail(Blog blog) =>
    context.pushNamed(RouteNames.blogDetail, pathParameters: {'slug': blog.slug});

// Detail screens use the shared extension so a deep link is not a dead end:
void _back() => context.popOrGoNamed(RouteNames.blogList);
```

Infinite scroll — the controller self-guards, so the listener stays dumb:

```dart
void _onScroll() {
  if (!_scrollController.hasClients) return;
  final position = _scrollController.position;
  if (position.pixels >= position.maxScrollExtent - _loadMoreExtent) {
    ref.read(blogListControllerProvider.notifier).loadMore();
  }
}
```

**Theme**: prefer `Theme.of(context).textTheme…` then override colours from
`lib/shared/theme/app_colors.dart` (`AppColors.primary` is the WEUP red `0xFFC0181E`). Never
redeclare a brand colour in a feature, and never build a `ThemeData` inside one. Only a light theme
exists. Sizing uses `flutter_screenutil` `.w/.h/.r` against a 375×812 design.

**Logging**: no `print` (lint error). Use `AppLogger.debug/info/warning/error(event, category:,
context:)`, which redacts a wide sensitive-key regex.

---

## 12. Register the route

Two files, sometimes three.

```dart
// lib/app/router/route_names.dart
static const String blogList = 'blog_list';
static const String blogDetail = 'blog_detail';
```

```dart
// lib/app/router/app_router.dart — nested under /home inside the ShellRoute
GoRoute(
  path: 'blog',
  name: RouteNames.blogList,
  builder: (context, state) => const BlogListScreen(),
  routes: [
    GoRoute(
      path: ':slug',
      name: RouteNames.blogDetail,
      builder: (context, state) {
        final slug = state.pathParameters['slug'] ?? '';
        return BlogDetailScreen(slug: slug);
      },
    ),
  ],
),
```

**Shell or root?** A `ShellRoute` keeps the bottom nav and the persistent audio mini player. Routes
that should take over the screen — readers, players, auth — sit outside it with
`parentNavigatorKey: rootNavigatorKey`.

**The shell decides bottom-nav visibility, not the route:**

```dart
// lib/app/shell/main_navigation_shell.dart
const _tabPaths = <String>{'/home', '/home/blog', '/explore', '/library', '/cart', '/profile'};
```

Any in-shell route **not** in that set hides the bar but keeps the mini player. Adding a real tab
means updating `_tabPaths`, `_selectedIndex(location)` and `_onItemTapped`.

For enum path params, use `tryParse` with a fallback so a bad deep link degrades instead of throwing:
`ProductType.tryParse(rawType) ?? ProductType.book`.

## 13. Translations — all 9 ARB files

```
lib/l10n/{app_en,app_en_US,app_es,app_fil,app_id,app_ms,app_pt,app_th,app_vi}.arb
```

Keys are lowerCamelCase with a feature prefix: `blogSearchHint`, `blogEmpty`, `blogLoadError`.
Placeholder and plural metadata goes in `app_en.arb` only:

```jsonc
"courseLessonCount": "{count, plural, =0{No lessons} =1{1 lesson} other{{count} lessons}}",
"@courseLessonCount": {
  "description": "…",
  "placeholders": { "count": { "type": "int" } }
}
```

Then `make gen`. Access is `AppLocalizations.of(context).<key>`, and **only in `presentation`** —
`domain`, `data` and `core` must not import `AppLocalizations` or take a `BuildContext` (L10N-7).

> **`tool/check_l10n.dart` fails the build** on a missing key, an extra key, a wrong `@@locale`, or
> an empty translation. There is **no parent-locale fallback** (L10N-3), so `app_en_US.arb` must
> carry every key too. This runs inside `make gen`, so you find out immediately.

`lib/l10n/app_localizations*.dart` is generated and gitignored — **never hand-edit it.**

## 14. Tests

Test files mirror the source tree. Write repository and controller tests **before** wiring the page
(TOOL-11).

Controller tests override the repository with a hand-written fake and keep a listener alive, because
the provider is `autoDispose`:

```dart
ProviderContainer makeContainer(BlogRepository repository) {
  final container = ProviderContainer(
    overrides: [blogRepositoryProvider.overrideWithValue(repository)],
  );
  container.listen(blogListControllerProvider, (previous, next) {});   // ← or it disposes mid-await
  addTearDown(container.dispose);
  return container;
}
```

Datasource and repository tests build a real `Dio` + `ApiClient` and swap
`dio.httpClientAdapter` for a callback adapter returning canned JSON — see
`test/features/blog/data/repositories/blog_repository_impl_test.dart`.

```bash
make check          # gen + format-check + analyze + test — run this before every commit
```

---

## 15. Conventions

| Kind | Pattern |
|---|---|
| Files | `snake_case.dart`, plural directories |
| DI root | `<feature>_providers.dart` |
| DTO | `<feature>_<x>_model.dart` → class `…Model` |
| Mapper | `<feature>_mapper.dart` → `extension …Mapper on <Model>` |
| Datasource | `<feature>_remote_data_source.dart` → `final class …RemoteDataSource` |
| Repo contract | `<feature>_repository.dart` → `abstract interface class …Repository` |
| Repo impl | `<feature>_repository_impl.dart` → `final class …RepositoryImpl` |
| Entity | `<noun>.dart` → **no suffix** |
| Controller | `<feature>_controller.dart` → `extends AsyncNotifier`/`Notifier` |
| Screen | `<feature>_screen.dart` → suffix `Screen` |
| Provider | suffix `Provider`, camelCase |

Class defaults: **`final class`** by default, contracts `abstract interface class`, closed unions
`sealed class`.

**Comment language is Vietnamese**; identifiers are English; user-facing text goes through ARB.

`ARCH-6` decides `shared/` vs per-feature: an endpoint hit by ≥ 2 features with *the same request and
response* belongs in `shared/data` + `shared/providers`. The same URL with *different DTO subsets*
shares only the path via `ApiEndpoints`.

`AGENTS.md` hard stop: **do not add a dependency to `pubspec.yaml` without asking.**

---

## 16. Platform notes

- **iOS**: `pod install` in `ios/` on first checkout. Google Sign-In is configured in
  `ios/Runner/Info.plist` via `GIDClientID`, `GIDServerClientID` and a reverse-client-id URL scheme —
  **not** a `GoogleService-Info.plist`. `UIBackgroundModes: [audio]` is already set. Debug/JIT is
  blocked on some physical devices; use `make run-profile`.
- **Android**: namespace `com.weup.elearning`, `applicationId` **`com.weup.book`**, JDK 17.
  **Release builds are currently signed with the debug key** — you need `android/key.properties`
  (gitignored) and a real keystore before shipping. See
  `docs/development-guide/PRODUCTION_READINESS.md` §D1.
- **Firebase is not used at all.** No `google-services.json`, no `GoogleService-Info.plist`, no
  crash reporter. `bootstrap.dart` writes structured local logs only.
- `lib/features/_feature_template/` is **not runtime code.** Copy it, delete its `README.md` and
  `.gitkeep` files, rename `example_model.dart`, then `make gen`.
- `lib/shared/mock/mock_data.dart` still backs some screens. `ARCHITECTURE.md` §14 lists the gaps.

---

## Checklist

```
[ ] make setup run (codegen is gitignored)
[ ] config/dev.local.json if pointing at a local backend
[ ] endpoint path added to lib/core/network/api_endpoints.dart   ← never inline it
[ ] datasource with _unwrapData validation + TypeError→FormatException
[ ] correct NetworkAuthMode (none for public)
[ ] new public /auth/* path? added to AuthInterceptor._publicAuthPaths
[ ] Freezed model, all fields nullable-or-defaulted, dates as String
[ ] mapper extension throwing FormatException on unrenderable data
[ ] entity: plain Dart, List.unmodifiable for collections
[ ] repository contract in domain/ returning Result<T>
[ ] repository impl with the _guard error mapping
[ ] <feature>_providers.dart typed as the INTERFACE
[ ] controller with retry: null + _requestVersion race guard
[ ] screen renders loading / data / empty / error+retry
[ ] RefreshIndicator on lists
[ ] navigation by RouteNames, never a literal
[ ] route in route_names.dart + app_router.dart
[ ] shell vs rootNavigatorKey decided
[ ] _tabPaths updated if it is a tab
[ ] keys in ALL 9 .arb files
[ ] api.md written for the feature
[ ] repository + controller tests
[ ] make check passes
[ ] verified on both an Android emulator and an iOS simulator
```
