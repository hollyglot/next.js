# Next.js Codebase Research Document

> **Date:** 2026-04-02
> **Scope:** Full codebase documentation of the Next.js monorepo as it exists today
> **Branch:** `hollyglot/full-codebase-docs` (based on canary at `d529a233e8`)

---

## Summary of Findings

The Next.js monorepo is a polyglot (TypeScript + Rust) pnpm workspace containing the core Next.js framework, ~19 published packages, ~10 Rust crates, the Turbopack bundler (git subtree), 100+ example apps, and a comprehensive test suite. The codebase supports three bundlers (Turbopack, Webpack, Rspack), two routing systems (App Router with React Server Components, Pages Router), and two server runtimes (Node.js, Edge/Web). The architecture is layered: CLI commands dispatch to server classes that route requests through matcher pipelines to route modules, which delegate to rendering pipelines (RSC Flight or Fizz HTML streaming), with caching at multiple levels (response cache, incremental cache, segment cache). The build system generates webpack/turbopack configurations, discovers routes via filesystem conventions, performs static analysis, and produces manifests consumed at runtime.

---

## Table of Contents

1. [Monorepo Structure](#1-monorepo-structure)
2. [Core Package: packages/next](#2-core-package-packagesnext)
3. [Server Runtime](#3-server-runtime)
4. [App Router Rendering Pipeline](#4-app-router-rendering-pipeline)
5. [Client Runtime](#5-client-runtime)
6. [Build System](#6-build-system)
7. [CLI Commands](#7-cli-commands)
8. [Shared Libraries](#8-shared-libraries)
9. [Other Packages](#9-other-packages)
10. [Rust Crates & Turbopack](#10-rust-crates--turbopack)
11. [Configuration & Infrastructure](#11-configuration--infrastructure)
12. [Testing Infrastructure](#12-testing-infrastructure)
13. [Cross-Component Data Flows](#13-cross-component-data-flows)

---

## 1. Monorepo Structure

```
next.js/
├── packages/              # ~19 published npm packages
│   ├── next/              # Core framework (published as `next`)
│   ├── create-next-app/   # Project scaffolding CLI
│   ├── next-swc/          # Native Rust/SWC bindings
│   ├── eslint-plugin-next/# ESLint rules
│   ├── font/              # next/font implementation
│   ├── third-parties/     # Google Analytics, GTM, YouTube, Maps
│   ├── next-codemod/      # 77 migration codemods
│   ├── next-bundle-analyzer/ # Webpack bundle analysis wrapper
│   ├── next-mdx/          # MDX integration
│   ├── next-env/          # Dotenv loading with priority ordering
│   ├── next-routing/      # Route resolution engine (experimental)
│   ├── next-rspack/       # Rspack bundler integration (experimental)
│   ├── next-playwright/   # Playwright testing utilities
│   ├── react-refresh-utils/# React Fast Refresh for dev HMR
│   ├── eslint-config-next/# ESLint config preset
│   └── eslint-plugin-internal/ # Internal ESLint rules
├── crates/                # Rust crates for SWC/Next.js bindings
├── turbopack/             # Turbopack bundler (git subtree, 55+ crates)
├── rspack/                # Rspack integration (separate workspace)
├── apps/                  # Internal apps (docs, bundle-analyzer UI)
├── bench/                 # 10+ benchmark applications
├── test/                  # All test suites (e2e, dev, prod, unit)
├── examples/              # 100+ example Next.js applications
├── docs/                  # Documentation (migrating to apps/docs)
├── errors/                # Error documentation system
├── scripts/               # Build, release, profiling scripts
├── evals/                 # Evaluation suites
├── contributing/          # Contributing docs
└── patches/               # 7 dependency patches (webpack-sources, etc.)
```

**Workspace configuration:** `pnpm-workspace.yaml` includes `packages/*`, `apps/*`, `bench/*`, `crates/*/js`, `turbopack/crates/*/js`, `turbopack/packages/*`.

**Build orchestration:** `turbo.json` defines task pipeline with `build`, `dev`, `typescript` tasks, global env `NEXT_CI_RUNNER`, TUI enabled.

**Rust workspace:** `Cargo.toml` at root manages all Next.js crates + Turbopack crates via glob `turbopack/crates/*`. Custom profiles: `dev` (line-tables-only), `release` (thin LTO).

---

## 2. Core Package: packages/next

The main Next.js framework. Source is in `packages/next/src/`, compiled output in `packages/next/dist/`.

### Source Directory Layout

```
packages/next/src/
├── cli/           # CLI command entry points
├── server/        # Server runtime (~150 files)
├── client/        # Client runtime (routing, hydration, components)
├── build/         # Build system (~263 files)
├── lib/           # Shared utilities, config loading, metadata
├── shared/        # Cross-cutting types, constants, router utils
├── api/           # Public API re-exports (next/dynamic, next/headers, etc.)
├── export/        # Static export (output: export)
├── telemetry/     # Anonymous usage telemetry
├── trace/         # OpenTelemetry-compatible tracing
├── diagnostics/   # Build diagnostics recording
├── experimental/  # Experimental features (testmode, playwright)
├── compiled/      # Pre-compiled dependencies
├── bundles/       # Bundle configurations
├── bin/           # Binary entry points
├── pages/         # Built-in pages (_app, _document, _error)
├── next-devtools/ # Development tools
└── types.ts       # Public type definitions
```

### Key Entry Points

| Entry             | File                                                        | Purpose                |
| ----------------- | ----------------------------------------------------------- | ---------------------- |
| Dev server        | `src/cli/next-dev.ts` → `src/server/dev/next-dev-server.ts` | Development with HMR   |
| Production server | `src/cli/next-start.ts` → `src/server/next-server.ts`       | Production serving     |
| Build             | `src/cli/next-build.ts` → `src/build/index.ts`              | Production compilation |

---

## 3. Server Runtime

**Location:** `packages/next/src/server/`

### Server Class Hierarchy

```
BaseServer (base-server.ts:~300)
  ├── NextNodeServer (next-server.ts:175)
  │   └── DevServer (dev/next-dev-server.ts:122)
  └── (Edge runtime uses web/adapter.ts instead)
```

#### BaseServer (`base-server.ts:~300`)

- Abstract base class for all server implementations
- Key types: `FindComponentsResult<T>` (:161), `MiddlewareRoutingItem` (:168), `RouteHandler` (:174), `Options` (:196), `RenderOpts` (:244), `BaseRequestHandler` (:268)
- Responsibilities: route matching/resolution, component loading/caching, middleware routing, request normalization, response caching, i18n handling, prerender manifest management

#### NextNodeServer (`next-server.ts:175`)

- Extends BaseServer for Node.js production runtime
- Key exports: `NodeRequestHandler` type (:144)
- Properties: `middlewareManifestPath` (:180), `imageResponseCache` (:182), `dynamicRoutes` (:185), `cleanupListeners` (:195), `internalWaitUntil` (:196)
- Constructor (lines 200-295): initializes middleware, caching, environment

#### DevServer (`dev/next-dev-server.ts:122`)

- Extends NextNodeServer with development features
- Properties: `bundlerService` (:137), `serverComponentsHmrCache` (:142), `staticPathsWorker` (:146)
- Three HMR implementations: `hot-reloader-webpack.ts`, `hot-reloader-turbopack.ts`, `hot-reloader-rspack.ts`
- On-demand entry handler for lazy route compilation

### Request/Response Abstraction

```
BaseNextRequest (base-http/index.ts:28)     BaseNextResponse (base-http/index.ts:47)
    │                                            │
    └── NodeNextRequest (base-http/node.ts:19)   └── NodeNextResponse (base-http/node.ts:75)
```

### Router & Route Matching

**Router Server** (`lib/router-server.ts:82`): Entry point for request routing. `initialize(opts)` loads config, sets up filesystem checker, compression, and (in dev) the bundler.

**Route matching pipeline:**

- `route-definitions/` — Route definition types
- `route-matchers/` — Pattern matching: `app-page-route-matcher.ts`, `app-route-route-matcher.ts`, `pages-route-matcher.ts`, `pages-api-route-matcher.ts`, `locale-route-matcher.ts`
- `route-matcher-managers/` — Orchestration: `default-route-matcher-manager.ts` (prod), `dev-route-matcher-manager.ts` (dev)
- `route-matcher-providers/` — Create matchers from manifests

### Route Modules

```
RouteModule (route-modules/route-module.ts:105) — abstract base
  ├── AppPageRouteModule (route-modules/app-page/module.ts:82)
  ├── AppRouteModule (route-modules/app-route/module.ts:~150)
  ├── PagesRouteModule (route-modules/pages/module.ts)
  └── PagesAPIRouteModule (route-modules/pages-api/module.ts)
```

- **AppPageRouteModule**: `match()` (:90), `normalizeUrl()` (:113), `render()` (:150+) — delegates to app-render pipeline
- **AppRouteModule**: HTTP method dispatch, dynamic payload handling, edge runtime support

### Caching Layer

**ResponseCache** (`response-cache/index.ts:107`):

- In-memory LRU cache with batching for coalesced requests
- Config: `NEXT_PRIVATE_RESPONSE_CACHE_TTL` (default 10s), `NEXT_PRIVATE_RESPONSE_CACHE_MAX_SIZE` (default 150)

**IncrementalCache** (`lib/incremental-cache/index.ts:84`):

- ISR/static revalidation caching
- `CacheHandler` abstract interface (:59) — default: `FileSystemCache`
- Tag-based revalidation, stale-while-revalidate, custom handlers via `Symbol.for('@next/cache-handlers')`
- Cache handler configuration: `cacheHandlers: { default?, remote?, static? }` in NextConfig

### Async Storage & Context

| Storage              | File                                                | Scope                                                                                        |
| -------------------- | --------------------------------------------------- | -------------------------------------------------------------------------------------------- |
| WorkAsyncStorage     | `app-render/work-async-storage.external.ts:15`      | Request lifetime — `WorkStore` with `isStaticGeneration`, `incrementalCache`, `afterContext` |
| WorkUnitAsyncStorage | `app-render/work-unit-async-storage.external.ts:36` | Per-work-unit — `RequestStore` with `url`, `headers`, `cookies`, `phase`                     |
| ActionAsyncStorage   | `app-render/action-async-storage.external.ts`       | Server action execution context                                                              |

### After API (`after/`)

```typescript
// after/after.ts:9
export function after<T>(task: AfterTask<T>): void
```

- `AfterContext` (`after-context.ts:21`): Queues callbacks, wraps in `bindSnapshot()` to preserve ALS context
- `AfterRunner` (`run-with-after.ts`): `waitUntil()` → `onClose()` → `executeAfter()`
- Phase changes to `'after'` when running tasks post-response

### Edge/Web Runtime (`web/`)

- **Adapter** (`web/adapter.ts:109`): Bridges Node and Edge runtimes, creates `NextRequestHint` + `NextFetchEvent`, runs in async storage contexts
- **EdgeRouteModuleWrapper** (`web/edge-route-module-wrapper.ts:35`): Wraps `AppRouteRouteModule` for edge runtime
- **Sandbox** (`web/sandbox/sandbox.ts`): Isolated edge function execution
- **Spec extensions** (`web/spec-extension/`): `NextRequest`, `NextResponse`, `cookies`, `headers`, `fetch-event`, `image-response`, `user-agent`

### Normalizers (`normalizers/`)

Path normalization pipeline: RSC normalizer, NextData normalizer, SegmentPrefixRSC normalizer, base-path normalizer, locale normalizer.

### Configuration

- **`config-shared.ts`**: `NextConfig` interface (:1147), `NextConfigComplete` (:27), `ExperimentalConfig` (:395, 100+ flags), `defaultConfig` (:1624, frozen), `normalizeConfig()` (:1820), `NextConfigRuntime` (:1841)
- **`config.ts`**: `loadConfig()` (:1527-1543) — cache check → standalone config → load env → find config file → import → normalize → validate (Zod) → apply modifiers → cache

Key experimental flags: `ppr` (Partial Prerendering), `cacheComponents`, `staleTimes`, `dynamicOnHover`, `serverActions`, `authInterrupts`, `prefetchInlining`

---

## 4. App Router Rendering Pipeline

**Location:** `packages/next/src/server/app-render/`

### Request Flow

```
HTTP Request
  → web/adapter.ts (create NextRequestHint + ALS contexts)
  → AppPageRouteModule.render()
  → app-render.tsx:2691 renderToHTMLOrFlight()
    → workAsyncStorage.run() (establish WorkStore scope)
    → app-render.tsx:2205 renderToHTMLOrFlightImpl()
      → app-render.tsx:752 generateDynamicFlightRenderResult()
        → app-render.tsx:565 generateDynamicRSCPayload()
          → create-component-tree.tsx:51 createComponentTree()
          → Build Flight data structure (RSCPayload)
        → stream-ops.web.ts renderToFlightStream()
          → react-server-dom-webpack/server.renderToReadableStream()
      → FlightRenderResult (RSC) OR Fizz HTML response
  → Stream to response
  → after-context.ts runCallbacks() (post-response tasks)
```

### Key Files

| File                        | Key Export                                                                                             | Line   |
| --------------------------- | ------------------------------------------------------------------------------------------------------ | ------ |
| `app-render.tsx`            | `renderToHTMLOrFlight()`                                                                               | :2691  |
| `app-render.tsx`            | `renderToHTMLOrFlightImpl()`                                                                           | :2205  |
| `app-render.tsx`            | `generateDynamicFlightRenderResult()`                                                                  | :752   |
| `app-render.tsx`            | `generateDynamicRSCPayload()`                                                                          | :565   |
| `create-component-tree.tsx` | `createComponentTree()`                                                                                | :51    |
| `entry-base.ts`             | React Server layer exports (`renderToReadableStream`, `decodeReply`, `LayoutRouter`, etc.)             | :1-131 |
| `stream-ops.web.ts`         | `renderToFlightStream()`, `renderToFizzStream()`, `continueFizzStream()`, `continueDynamicPrerender()` | —      |
| `dynamic-rendering.ts`      | `trackAllowedDynamicAccess()`, `consumeDynamicAccess()`, `createDynamicTrackingState()`                | —      |
| `postponed-state.ts`        | PPR resumption data                                                                                    | —      |
| `create-error-handler.ts`   | `createReactServerErrorHandler()`, `createHTMLErrorHandler()`                                          | —      |
| `flight-render-result.ts`   | `FlightRenderResult` (wraps stream with RSC content type)                                              | :1-20  |

### Pages Router Rendering

**`render.tsx`**: `renderToString()` (:137), `ServerRouter` class (:143), `renderToHTML` export. Handles `getServerSideProps`, `getStaticProps`, Document/page composition, HTML streaming.

### Stream Utilities (`stream-utils/`)

- `encoded-tags.ts` — Uint8Array constants for HTML tags used in streaming
- `node-web-streams-helper.ts` — Bridge between Node.js Readable and Web ReadableStream; `streamToString()`, `chainStreams()`, `renderToInitialFizzStream()`
- `uint8array-helpers.ts` — Byte array manipulation

---

## 5. Client Runtime

**Location:** `packages/next/src/client/`

### App Router Client Architecture

```
AppRouter (app-router.tsx:571-600)
  └── Router (app-router.tsx:154-569)
      ├── HistoryUpdater (:59-112) — syncs browser history
      ├── Context Providers:
      │   ├── AppRouterContext → publicAppRouterInstance
      │   ├── LayoutRouterContext → tree/cache/segments
      │   ├── GlobalLayoutRouterContext → tree/focusAndScrollRef/nextUrl
      │   ├── SearchParamsContext → URLSearchParams
      │   ├── PathnameContext → pathname string
      │   ├── PathParamsContext → params object
      │   └── NavigationPromisesContext → (dev-only)
      ├── RedirectBoundary — catches redirect() errors
      ├── RootLayoutBoundary — error detection
      ├── HotReloader (dev only) — error overlay + HMR
      └── RootErrorBoundary — catches all errors
```

### Router State Machine

**State type** (`router-reducer-types.ts:1-264`):

- `tree` — `FlightRouterState` representing UI structure
- `cache` — `CacheNode` with RSC data and prefetch data
- `pushRef` — History entry control (pendingPush, mpaNavigation)
- `focusAndScrollRef` — Focus/scroll management
- `canonicalUrl` — Browser-visible URL
- `nextUrl` — Internal URL for intercepting routes

**Actions:**
| Action | Type | Purpose |
|--------|------|---------|
| `ACTION_NAVIGATE` | `NavigateAction` | Navigate to new URL |
| `ACTION_RESTORE` | `RestoreAction` | Restore from history/BFCache |
| `ACTION_SERVER_PATCH` | `ServerPatchAction` | Server-side route mismatch |
| `ACTION_REFRESH` | `RefreshAction` | Refresh page data |
| `ACTION_HMR_REFRESH` | `HmrRefreshAction` | HMR refresh in dev |
| `ACTION_SERVER_ACTION` | `ServerActionAction` | Server action execution |

**Reducer** (`router-reducer.ts`): `clientReducer()` dispatches to sub-handlers in `reducers/` — `navigate-reducer.ts`, `server-patch-reducer.ts`, `refresh-reducer.ts`, `restore-reducer.ts`, `hmr-refresh-reducer.ts`, `server-action-reducer.ts`.

### Action Queue (`app-router-instance.ts:1-150`)

- `dispatchAction()` — Queues router actions for sequential processing
- `dispatchNavigateAction()` — Navigates to URL with prefetch data
- `publicAppRouterInstance` — Public `AppRouter` API instance
- Actions queue and execute sequentially; supports async actions

### Link Component & Prefetching

**Link** (`app-dir/link.tsx:337-771`):

- Props: `href`, `prefetch` (boolean | 'auto' | null), `scroll`, `replace`, `onNavigate`, `transitionTypes`
- IntersectionObserver for visibility-based prefetching
- `mountLinkInstance()` registers with prefetch system
- `useLinkStatus()` hook (:777) for optimistic pending state

**Prefetch system** (`components/segment-cache/`):

- `scheduler.ts` — Background task scheduler with priorities: `Intent` (2, hover/touch), `Default` (1, visible), `Background` (0, revalidation)
- `prefetch.ts` — Data fetching logic
- Fetch strategies: `LoadingBoundary` (0), `PPR` (1), `PPRRuntime` (2), `Full` (3)

### Segment Cache (`components/segment-cache/`)

| File            | Purpose                                                                                                                            |
| --------------- | ---------------------------------------------------------------------------------------------------------------------------------- |
| `cache.ts`      | Route and segment data storage; `RouteCacheEntry`, `SegmentCacheEntry`, `readOrCreateRouteCacheEntry()`, `fetchRouteOnCacheMiss()` |
| `cache-key.ts`  | Cache key generation: `RouteCacheKey` from URL/state                                                                               |
| `navigation.ts` | Navigate by querying cache: `navigate()` (:51), `navigateImpl()`, `completeHardNavigation()`                                       |
| `vary-path.ts`  | Tracks which request parts affect response (headers, cookies)                                                                      |

### BFCache (`bfcache-state-manager.ts`)

- `useRouterBFCache()` hook (:38-120) — Preserves React state on back/forward
- Linked list of 1-3 `RouterBFCacheEntry` entries (configurable `MAX_BF_CACHE_ENTRIES`)
- Entries hold `tree`, `cacheNode`, `stateKey`; passed to `<Activity>` boundaries

### Error Handling

| Component              | File                             | Purpose                                                                         |
| ---------------------- | -------------------------------- | ------------------------------------------------------------------------------- |
| `ErrorBoundaryHandler` | `error-boundary.tsx:42`          | Catches render errors, resets on pathname change, detects `isNextRouterError()` |
| `RedirectBoundary`     | `redirect-boundary.tsx:38-79`    | Catches `redirect()` errors, triggers `router.push/replace`                     |
| `DefaultGlobalError`   | `builtin/global-error.tsx`       | Fallback error UI with reload/back buttons                                      |
| `RootErrorBoundary`    | `errors/root-error-boundary.tsx` | Final fallback wrapping entire app                                              |

### Hydration (`app-index.tsx`)

1. Server renders HTML with `window.__next_f = []` global Flight array
2. Inline scripts push Flight segments: type 0 (bootstrap), 1 (data), 2 (form state), 3 (binary)
3. `nextServerDataCallback()` (:79-110) consumes chunks into `ReadableStream`
4. `createFromReadableStream()` feeds to React → `ReactDOM.hydrateRoot()`
5. `DOMContentLoaded` finalizes stream

### Other Client Components

- **LayoutRouter** (`layout-router.tsx`): Per-segment rendering with caching, scroll/focus handling, `<Activity>` boundaries for BFCache
- **ClientPageRoot** (`client-page.tsx:18-72`): Wraps client pages with server-provided params/searchParams
- **Image** (`image-component.tsx`): Optimized image with srcset, lazy loading, blur placeholders
- **Script** (`script.tsx`): Script loading strategies: `beforeInteractive`, `afterInteractive`, `lazyOnload`, `worker`
- **Form** (`form.tsx`): Form wrapper for server actions with navigation integration
- **Pages Router** (`router.ts`, `index.tsx`, `link.tsx`): Legacy singleton router with events and code splitting

### Navigation Hooks

- `useRouter()` — App router instance (`AppRouterInstance`)
- `useSearchParams()` — Current search params (readonly)
- `usePathname()` — Current pathname
- `useParams()` — Current route params
- All from `components/navigation.ts`

---

## 6. Build System

**Location:** `packages/next/src/build/` (~263 TypeScript files)

### Main Build Entry (`index.ts`, 156 KB)

`export default async function build(...)` orchestrates the entire pipeline:

1. **Initialization** (:951-1071): `loadEnvConfig()`, `loadConfig()`, validate `deploymentId`, `installBindings()` (SWC)
2. **Route Discovery** (:1079-1492): `loadCustomRoutes()`, `discoverRoutes()`, `createPagesMapping()`, validate app paths, detect middleware
3. **Type Generation** (:1390-1426): `createRouteTypesManifest()`, write validator file, cache life types
4. **Manifest Generation** (:1518-1700): `generateRoutesManifest()`, `getStaticInfoIncludingLayouts()`
5. **Static Prerendering** (:1750+): Prerender static routes, collect static paths from `generateStaticParams()`
6. **Bundler Execution** (:1800+): Select Turbopack or Webpack, run bundler, write traces/manifests

Key types: `PrerenderManifestRoute` (:233), `PrerenderManifest` (:378), `RoutesManifest` (:432), `FunctionsConfigManifest` (:592)

### Bundler Integration

**Turbopack** (`turbopack-build/`):

- `index.ts:78` — `turbopackBuild(withWorker)` spawns worker, loads Rust bindings
- `impl.ts` — Actual Turbopack implementation

**Webpack** (`webpack-build/`):

- `index.ts:131` — `webpackBuild(withWorker, compilerNames)` runs 3 compilers: server → edge-server → client
- `compiler.ts:39` — `runCompiler(config, opts)` creates webpack compiler

### Webpack Configuration (`webpack-config.ts`, 101 KB)

Generates webpack config objects for all three compiler targets. Key patterns:

- `attachReactRefresh()` (:187) — React Fast Refresh loader
- `babelIncludeRegexes` (:121-151) — Modules to transpile
- `EXTERNAL_PACKAGES` — from `server-external-packages.jsonc` (:107)

### Webpack Loaders (22+)

| Loader                               | Purpose                        |
| ------------------------------------ | ------------------------------ |
| `next-app-loader/index.ts`           | App Router pages (2500+ lines) |
| `next-client-pages-loader.ts`        | Pages Router client pages      |
| `next-swc-loader.ts`                 | SWC transpilation              |
| `next-flight-client-entry-loader.ts` | RSC client entry points        |
| `next-flight-action-entry-loader.ts` | Server Actions entry           |
| `next-flight-css-loader.ts`          | CSS from RSC                   |
| `next-middleware-loader.ts`          | Middleware entry               |
| `next-edge-function-loader.ts`       | Edge function                  |
| `next-metadata-route-loader.ts`      | Metadata routes                |
| `next-barrel-loader.ts`              | Barrel file optimization       |
| `modularize-import-loader.ts`        | modularizeImports              |

### Webpack Plugins (45+)

**Manifests:** `build-manifest-plugin.ts`, `pages-manifest-plugin.ts`, `flight-manifest-plugin.ts`, `flight-client-entry-plugin.ts`, `next-font-manifest-plugin.ts`

**Optimization:** `deferred-entries-plugin.ts`, `css-chunking-plugin.ts`, `memory-with-gc-cache-plugin.ts`

**Error handling:** `wellknown-errors-plugin/` (7 files for Sass, Babel, fonts, CSS, imports, etc.)

**Other:** `subresource-integrity-plugin.ts`, `middleware-plugin.ts`, `next-trace-entrypoints-plugin.ts`, `telemetry-plugin/`

### Route Discovery (`route-discovery.ts`, 14.9 KB)

- `discoverRoutes()` — Main export, discovers all routes
- `collectAppFiles()` (:85) — Collects app directory files (pages, layouts, defaults)
- `collectPagesFiles()` (:120) — Collects pages directory files
- `getPageFromPath()` (:61) — Extracts page route from file path

### Static Analysis (`analysis/`)

- `get-page-static-info.ts` (26.3 KB): `getPageStaticInfo()` analyzes page exports (`runtime`, `preferredRegion`, `getStaticProps`, `generateStaticParams`, etc.)
- `extract-const-value.ts`: Extracts constant export values
- `parse-module.ts`: Parses module for exports

### Templates (`templates/`)

Code generation templates executed by loaders:

- `app-page.ts` (72.8 KB) — App page handler with `AppPageRouteModule`, PPR support
- `app-route.ts` (17.3 KB) — App API route handler
- `pages.ts` / `pages-api.ts` — Pages Router templates
- `middleware.ts` — Middleware entry point
- `edge-ssr-app.ts` / `edge-app-route.ts` — Edge runtime templates

### Entry Points (`entries.ts`, 23.8 KB)

`createEntrypoints()` (:~200) generates webpack entries for all routes. `getPageFilePath()` (:69) resolves paths handling `PAGES_DIR_ALIAS`, `APP_DIR_ALIAS`, `ROOT_DIR_ALIAS`.

### Static Paths (`static-paths/`)

- `app.ts` (41.7 KB) — App directory static path generation via `getStaticSegmentGenerators()`
- `pages.ts` (7.7 KB) — Pages directory static paths via `getStaticProps`
- Types: `PrerenderedRoute`, `FallbackRouteParam`

### Environment Variables (`define-env.ts`, 16.5 KB)

`getDefineEnv()` (:104) returns JSON-stringified webpack define replacements: `process.env.TURBOPACK`, `process.env.NODE_ENV`, `process.env.__NEXT_*`, image config, public env vars, i18n config, client router filters.

### SWC Integration (`swc/`)

- `index.ts` (1800+ lines): Platform detection, binary loading with WASM fallback
- `options.ts`: SWC compiler configuration
- `loaderWorkerPool.ts`: Worker pool for parallel transpilation
- `jest-transformer.ts`: SWC-based Jest transformer
- `generated-native.d.ts` / `generated-wasm.d.ts`: Binding type definitions

### Build Utilities

- `utils.ts` (47.6 KB): `detectConflictingPaths()`, `printTreeView()`, `copyTracedFiles()`, `collectRoutesUsingEdgeRuntime()`
- `build-context.ts` (3.1 KB): `NextBuildContext` shared state (dir, buildId, config, mappedPages)
- `create-compiler-aliases.ts` (27.4 KB): `createWebpackAliases()`, `createVendoredReactAliases()`, `createAppRouterApiAliases()`
- `handle-externals.ts` (13.9 KB): `makeExternalHandler()` — determines bundled vs external modules
- `collect-build-traces.ts` (21.4 KB): Module dependency traces for standalone builds
- `lockfile.ts` (9.2 KB): `Lockfile.acquireWithRetriesOrExit()` prevents concurrent builds

---

## 7. CLI Commands

**Location:** `packages/next/src/cli/`

| Command           | File                 | Purpose                                                  |
| ----------------- | -------------------- | -------------------------------------------------------- |
| `next dev`        | `next-dev.ts`        | Development server with HMR, Turbopack/Webpack selection |
| `next build`      | `next-build.ts`      | Production build with analysis/profiling                 |
| `next start`      | `next-start.ts`      | Production server on specified port/hostname             |
| `next export`     | `next-export.ts`     | Deprecated — redirects to `output: export` config        |
| `next lint`       | (via eslint)         | Linting with `@next/eslint-plugin-next`                  |
| `next info`       | `next-info.ts`       | System/dependency information                            |
| `next telemetry`  | `next-telemetry.ts`  | Enable/disable telemetry                                 |
| `next analyze`    | `next-analyze.ts`    | Bundle analysis reports                                  |
| `next post-build` | `next-post-build.ts` | Compact Turbopack cache                                  |
| `next typegen`    | `next-typegen.ts`    | Route type manifest generation                           |
| `next upgrade`    | `next-upgrade.ts`    | Spawns `@next/codemod` for migrations                    |
| `next test`       | `next-test.ts`       | Test runner wrapper (Playwright detection)               |

---

## 8. Shared Libraries

### `packages/next/src/lib/`

**Configuration:** `find-config.ts` (config file discovery), `find-pages-dir.ts` (pages/app dirs), `get-project-dir.ts` (project root with typo detection), `load-custom-routes.ts` (redirects/rewrites/headers)

**Metadata system** (`metadata/`): `resolve-metadata.ts`, resolvers for icons/opengraph/titles/URLs, `get-metadata-route.ts` (robots.txt, sitemap.xml)

**Bundler control:** `bundler.ts` — `parseBundlerArgs()` parses CLI flags/env vars to select Turbopack/Webpack/Rspack

**File operations:** `file-exists.ts`, `recursive-copy.ts`, `recursive-delete.ts`, `recursive-readdir.ts`

**Dependencies:** `has-necessary-dependencies.ts`, `install-dependencies.ts`, `download-swc.ts`

**Build utilities:** `constants.ts` (content-type/cache headers, middleware patterns), `worker.ts` (Jest worker pool for parallelization)

**TypeScript:** `typescript/runTypeCheck.ts`, `typescript/diagnosticFormatter.ts`

**Memory:** `memory/startup.ts`, `shutdown.ts`, `gc-observer.ts`, `trace.ts`

### `packages/next/src/shared/lib/`

**Runtime contexts** (`.shared-runtime.ts` files): `app-router-context`, `router-context`, `head-manager-context`, `hooks-client-context`, `html-context`, `loadable-context`, `image-config-context`

**Constants** (`constants.ts`): `COMPILER_NAMES`, adapter types, phase constants (`PHASE_PRODUCTION_BUILD`, `PHASE_DEVELOPMENT_SERVER`, etc.)

**Router utilities** (`router/utils/`): `app-paths.ts`, `is-dynamic.ts`, `normalize-page-path.ts`, `interception-routes.ts`, `format-url.ts`, `add-locale.ts`, `add-path-prefix.ts`

**Image handling:** `image-config.ts`, `image-loader.ts`, `image-blur-svg.ts`, `match-local-pattern.ts`, `match-remote-pattern.ts`

**Error classes** (`errors/`): `usage-error.ts`, `code-frame.ts`, `canary-only-config-error.ts`, `hard-deprecated-config-error.ts`

**Utilities:** `hash.ts` (fnv1a), `deep-freeze.ts`, `is-plain-object.ts`, `escape-regexp.ts`, `mitt.ts` (event emitter)

### `packages/next/src/api/` — Public API Re-exports

| File            | Export                               |
| --------------- | ------------------------------------ |
| `dynamic.ts`    | `next/dynamic`                       |
| `navigation.ts` | `useRouter`, `usePathname`, etc.     |
| `headers.ts`    | Request headers, cookies, draft-mode |
| `image.ts`      | Image component                      |
| `link.ts`       | Link component                       |
| `script.ts`     | Script component                     |
| `og.ts`         | OpenGraph image generation           |
| `form.ts`       | Form component                       |
| `error.ts`      | Error boundary component             |

### `packages/next/src/export/`

Static export system (`output: export`):

- `index.ts` — Main orchestrator, spawns workers, manages progress
- `worker.ts` — Worker process for parallel rendering
- `routes/app-page.ts` — Renders app directory pages (RSC)
- `routes/app-route.ts` — Exports app route handlers
- `routes/pages.ts` — Exports pages directory pages
- Types: `ExportAppOptions`, `ExportAppResult`, `ExportRouteResult`, `ExportPathEntry`

### `packages/next/src/telemetry/`

- `storage.ts` — `Telemetry` class: `isEnabled`, `record(event)`, `flush()`; uses `conf` package; `NEXT_TELEMETRY_DISABLED` env var
- Events: `events/build.ts`, `events/version.ts`, `events/plugins.ts`, `events/swc-load-failure.ts`, `events/mcp-telemetry.ts`

### `packages/next/src/trace/`

- `trace.ts` — `Span` class with nanosecond precision (`process.hrtime.bigint()`), threshold filtering via `NEXT_TRACE_SPAN_THRESHOLD_MS`
- `report/` — `to-json.ts`, `to-json-build.ts`, `to-telemetry.ts`
- `trace-uploader.ts` — Uploads traces to external service

### `packages/next/src/diagnostics/`

- `build-diagnostics.ts` — `recordFrameworkVersion()`, `updateBuildDiagnostics()`, `recordFetchMetrics()`
- Output files: `.next/diagnostics/build-diagnostics.json`, `fetch-metrics.json`, `framework.json`

### `packages/next/src/experimental/`

- **Testmode** (`testmode/`): Fetch/HTTP interception (`server.ts`), edge support (`server-edge.ts`), `AsyncLocalStorage` context, proxy server, Playwright integration with `nextFixture`, MSW integration
- **Server testing** (`testing/server/`): `unstable_getResponseFromNextConfig()` for testing redirects/rewrites/headers

---

## 9. Other Packages

### create-next-app (`packages/create-next-app/`)

- Entry: `index.ts`
- Interactive/non-interactive scaffolding, template selection (TypeScript/JavaScript, App/Pages Router, Tailwind), package manager detection, example support from GitHub
- Built with `@vercel/ncc` into single `dist/index.js`

### next-swc (`packages/next-swc/`)

- NAPI and WASM bindings to Next.js-customized SWC compiler
- Dependency flow: `next-custom-transforms` → `next-core` → `next-api` → napi/wasm bindings
- Feature flags: `plugin`, `image-extended` (webp/avif), `tokio-console`

### eslint-plugin-next (`packages/eslint-plugin-next/`)

- ~24 ESLint rules: font optimization, script safety, image optimization, component structure, HTML validation
- Configs: `recommended`, `core-web-vitals`

### font (`packages/font/`)

- Google Fonts: `src/google/loader.ts` (fetching, CSS generation, font axes)
- Local fonts: `src/local/loader.ts` (file processing, fallback metrics)
- Zero layout shift via `size-adjust` CSS property
- Private package compiled into `packages/next/dist/`, re-exported as `next/font`

### third-parties (`packages/third-parties/`)

- Components: `GoogleAnalytics`, `GoogleTagManager`, `GoogleMapsEmbed`, `YouTubeEmbed`
- Export via `@next/third-parties/google`

### next-codemod (`packages/next-codemod/`)

- 77 transform files using jscodeshift
- Major migrations: `cra-to-next.ts`, `metadata-to-viewport-export.ts`, `built-in-next-font.ts`

### next-env (`packages/next-env/`)

- `processEnv()` — loads `.env` files with priority: `.env.local` > `.env.{mode}.local` > `.env.{mode}` > `.env`
- Lazy loading, file change detection

### next-routing (`packages/next-routing/`)

- `resolveRoutes()` — Core route matching: beforeMiddleware → invokeMiddleware → beforeFiles → static → afterFiles → dynamicRoutes → fallback
- Regex source matching, header/cookie/query/host conditions, i18n support

### next-bundle-analyzer (`packages/next-bundle-analyzer/`)

- Lightweight wrapper around `webpack-bundle-analyzer`

### next-rspack (`packages/next-rspack/`)

- Experimental wrapper: `const withRspack = require('next-rspack'); module.exports = withRspack(nextConfig)`

### react-refresh-utils (`packages/react-refresh-utils/`)

- Integration between Turbopack/Webpack and `react-refresh` for HMR

---

## 10. Rust Crates & Turbopack

### Next.js Crates (`crates/`)

| Crate                        | Purpose                                                     | Key Modules                                                                                                                                                            |
| ---------------------------- | ----------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `next-napi-bindings`         | NAPI FFI bridge (`lib.rs:45-61`)                            | `code_frame`, `css`, `mdx`, `minify`, `next_api`, `parse`, `react_compiler`, `rspack`, `transform`, `turbopack`                                                        |
| `next-core`                  | High-level build abstractions (`lib.rs:7-42`)               | `app_structure` (70KB), `next_config` (68KB), `next_import_map` (68KB), `next_app`, `next_client`, `next_edge`, `next_font`, `next_image`, `next_pages`, `next_server` |
| `next-api`                   | Public Rust API (`lib.rs:6-30`)                             | `app.rs` (86KB), `pages.rs` (69KB), `module_graph.rs` (33KB), `middleware`, `entrypoints`, `routes_hashes_manifest`                                                    |
| `next-build`                 | Build orchestration                                         | Coordinates via `next-core` and turbopack                                                                                                                              |
| `next-custom-transforms`     | SWC visitor transforms (`lib.rs:40-73`) — no Turbopack deps | `chain_transforms`, `linter`, `react_compiler`, `transforms` (emotion, styled-jsx, relay, modularize imports, remove console)                                          |
| `next-code-frame`            | Error frame rendering with syntax highlighting              | `code_frame` CLI + library                                                                                                                                             |
| `next-taskless`              | Turbo-tasks-free utilities                                  | regex, serde_json, turbo-unix-path                                                                                                                                     |
| `next-error-code-swc-plugin` | Compile-time error code injection (cdylib)                  | md5, swc_core plugin                                                                                                                                                   |
| `wasm`                       | WASM bindings for browser/edge (cdylib)                     | next-custom-transforms, next-taskless, next-code-frame                                                                                                                 |

### Turbopack (`turbopack/`, git subtree)

55+ crates organized as:

**Infrastructure:** `turbo-tasks` (incremental computation), `turbo-tasks-backend`, `turbo-tasks-fs`, `turbo-tasks-fetch`, `turbo-tasks-env`, `turbo-tasks-hash`, `turbo-persistence`

**Core bundling:** `turbopack` (orchestration), `turbopack-core` (abstractions), `turbopack-cli`, `turbopack-dev-server`

**Language support:** `turbopack-ecmascript`, `turbopack-ecmascript-plugins`, `turbopack-ecmascript-runtime`, `turbopack-css`, `turbopack-mdx`, `turbopack-image`, `turbopack-wasm`

**Node.js:** `turbopack-node`, `turbopack-nodejs`, `turbopack-resolve`, `turbopack-nft`

---

## 11. Configuration & Infrastructure

### Root Config Files

| File                  | Purpose                                                     |
| --------------------- | ----------------------------------------------------------- |
| `package.json`        | pnpm workspace root, 356 lines, scripts for build/test/lint |
| `pnpm-workspace.yaml` | Workspace directories, update notifier disabled             |
| `turbo.json`          | Turbo task pipeline, global env, TUI enabled                |
| `Cargo.toml`          | Rust workspace, custom profiles                             |
| `tsconfig.json`       | Root TypeScript config with path aliases for test utils     |
| `.node-version`       | Node.js v20                                                 |
| `.npmrc`              | Auto-install peers, lockfile strict, provenance             |

### Linting & Formatting

| File                    | Purpose                                                                                       |
| ----------------------- | --------------------------------------------------------------------------------------------- |
| `eslint.config.mjs`     | Modern flat config: Babel parser, TypeScript ESLint, React/Hooks, Jest, Import, JSDoc plugins |
| `.prettierrc.json`      | Trailing comma es5, single quotes, no semicolons                                              |
| `lint-staged.config.js` | Pre-commit: prettier + eslint for JS/TS, rustfmt for Rust                                     |
| `.husky/pre-commit`     | Runs `pnpm lint-staged`                                                                       |
| `.typos.toml`           | Spell check (Rust files, custom words)                                                        |

### Patches (`patches/`)

Applied to: `webpack-sources@3.2.3`, `stacktrace-parser@0.1.10`, `@types/node@20.17.6`, `taskr@1.1.0`, `minizlib@3.1.0`, `http-proxy@1.18.1`, `@base-ui-components/react@1.0.0-beta.1`

### CI/CD (`.github/workflows/`)

Major workflows: `build_and_test.yml`, `build_and_deploy.yml`, `turbopack-nextjs-dev-integration-tests.yml`, `turbopack-nextjs-build-integration-tests.yml`, `rspack-nextjs-dev-integration-tests.yml`, `rspack-nextjs-build-integration-tests.yml`, `create_release_branch.yml`, `trigger_release.yml`, `deploy_docs.yml`, `test_examples.yml`, issue management workflows

### Scripts (`scripts/`)

**Build:** `build-native.ts`, `pack-next.ts`, `patch-next.ts`
**Testing:** `pr-status.js` (PR/CI analysis), `profile-next-dev-boot.js`, `benchmark-next-dev-boot.js`
**Release:** `publish-release.js`, `publish-native.js`, `create-release-branch.js`, `start-release.js`, `create-preview-tarballs.js`
**Maintenance:** `sync-react.js`, `update-google-fonts.js`, `validate-externals-doc.js`
**Tracing:** `trace-cli-startup.js`, `trace-next-server.js`, `trace-dd.mjs`, `analyze-dev-server-bundle.js`

---

## 12. Testing Infrastructure

### Test Directory Structure

```
test/
├── e2e/              # End-to-end tests (browser-based)
├── development/      # Dev server tests
├── production/       # Production build+start tests
├── unit/             # Unit tests (fast, no browser)
├── integration/      # Integration tests
├── lib/              # Test utilities and helpers
└── *.manifest.json   # Test manifests for different modes
```

### Test Modes

| Command                   | Mode                 | Bundler   |
| ------------------------- | -------------------- | --------- |
| `pnpm test-dev-turbo`     | Development          | Turbopack |
| `pnpm test-dev-webpack`   | Development          | Webpack   |
| `pnpm test-start-turbo`   | Production           | Turbopack |
| `pnpm test-start-webpack` | Production           | Webpack   |
| `pnpm test-unit`          | Unit tests           | N/A       |
| `pnpm testheadless`       | Headless, no rebuild | Current   |

### Test Framework

- **Jest** (`jest.config.js`): Global setup, verbose output, root `test/`, module path mapping, optional JUnit reporter for CI
- **`nextTestSetup()`**: Primary test harness — creates Next.js app instances for testing
- **`retry()`** from `next-test-utils`: Polling/waiting utility (replaces deprecated `check()`)
- **`pnpm new-test`**: Generates test scaffolding (`-- --args <appDir> <name> <type>`)

### Test Manifests

Generated manifests for different bundler/mode combinations:

- `turbopack-dev-tests-manifest.json`, `turbopack-build-tests-manifest.json`
- `rspack-dev-tests-manifest.json`, `rspack-build-tests-manifest.json`
- `cache-components-tests-manifest.json`, `deploy-tests-manifest.json`

### Key Environment Variables

| Variable                    | Purpose                                           |
| --------------------------- | ------------------------------------------------- |
| `NEXT_SKIP_ISOLATE=1`       | Skip packing Next.js for each test (~100s faster) |
| `NEXT_TEST_MODE=dev\|start` | Run dev or production mode                        |
| `IS_WEBPACK_TEST=1`         | Force webpack (turbopack is default)              |
| `HEADLESS=true`             | Run tests headless                                |

---

## 13. Cross-Component Data Flows

### Full Request Lifecycle (App Router)

```
Browser clicks <Link>
  → Client: dispatchNavigateAction() (app-router-instance.ts)
  → Client: navigate() queries segment cache (segment-cache/navigation.ts)
  → Client: If cache miss → fetch RSC from server
  → Server: router-server.ts routes request
  → Server: Route matcher finds AppPageRouteModule
  → Server: AppPageRouteModule.render()
  → Server: renderToHTMLOrFlight() (app-render.tsx:2691)
  → Server: workAsyncStorage.run() establishes request scope
  → Server: createComponentTree() builds React tree from LoaderTree
  → Server: renderToFlightStream() serializes RSC via react-server-dom-webpack
  → Server: ResponseCache / IncrementalCache check/store
  → Server: Stream Flight response
  → Client: Deserialize Flight → update router state → React re-renders
  → Client: HistoryUpdater calls history.pushState()
  → Client: Scroll/focus management
  → Server: after() callbacks execute post-response
```

### Build → Runtime Connection

```
Build Phase:
  build/index.ts → discoverRoutes() → createEntrypoints()
  → webpack-config.ts generates config → loaders transform source
  → templates/ injected with route data → plugins generate manifests
  → Output: .next/build-manifest.json, routes-manifest.json,
            pages-manifest.json, prerender-manifest.json,
            middleware-manifest.json, app-paths-manifest.json

Runtime Phase:
  next-server.ts reads manifests at startup
  → Route matchers built from manifest data
  → IncrementalCache initialized with prerender-manifest
  → ResponseCache wraps IncrementalCache
  → Requests matched against manifest-derived routes
  → LoadComponents() loads compiled route modules from dist/
```

### Config Flow

```
next.config.js (user)
  → config.ts:loadConfig() loads/normalizes/validates
  → config-shared.ts:NextConfigComplete (fully resolved)
  → build/define-env.ts injects into webpack defines (process.env.__NEXT_*)
  → Server reads at startup via NextConfigComplete
  → Client receives subset via serialized Flight data
  → Edge receives NextConfigRuntime (minimal subset)
```

### Caching Hierarchy

```
Client:
  Segment Cache (segment-cache/cache.ts)
    → RouteCacheEntry (route trees + segments)
    → SegmentCacheEntry (individual segment RSC data)
    → BFCache (bfcache-state-manager.ts, 1-3 entries)

Server:
  ResponseCache (response-cache/index.ts)
    → In-memory LRU (TTL: 10s, max: 150 entries)
    → Request batching/coalescing

  IncrementalCache (lib/incremental-cache/index.ts)
    → CacheHandler interface (pluggable)
    → Default: FileSystemCache (.next/cache/)
    → Custom: Redis, etc. via cacheHandlers config
    → Tag-based revalidation
    → Stale-while-revalidate
```
