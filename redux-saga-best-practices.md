# Redux + redux-saga: Production Best Practices & Audit Prompt

> Standards version: 2 (2026-10-05). What changed: see `STANDARDS-INDEX.md` → Version history.

Two parts:

1. **Best practices** — the rules, tiered as **MUST** (new code is blocked without it) and **SHOULD** (recommended, apply when touching the area). Rules tagged **[RN]** apply only to React Native; everything else applies to React web and React Native alike. Rules tagged **[C]** apply only to the DB-centric profile of §9.
2. **Audit prompt** — paste into Claude Code (or any agentic coding tool) in a project using Redux + redux-saga. It audits the project against part 1, writes/extends documentation, leaves a migration plan file, estimates effort, and asks before changing code.

Assumes TypeScript and Redux Toolkit (RTK). If a project is plain Redux without RTK, migrating to `configureStore`/`createSlice` is itself the first MUST.

---

## Part 1 — Best practices

### 1. Store setup

| Tier | Rule |
|---|---|
| MUST | Use `configureStore` from RTK, not `createStore`. Saga middleware is added via `getDefaultMiddleware({ thunk: false }).concat(sagaMiddleware)` unless thunks are deliberately used too (RTK 2 requires `middleware` to be a callback). A type cast on the saga middleware is an accepted, documented workaround until redux-saga ships RTK-2-compatible types. |
| MUST | Export `RootState = ReturnType<typeof store.getState>` and `AppDispatch = typeof store.dispatch` from the store module. |
| MUST | Export typed hooks `useAppDispatch` / `useAppSelector` (`useDispatch.withTypes<AppDispatch>()`, `useSelector.withTypes<RootState>()`; or `TypedUseSelectorHook` on react-redux < 9.1) from one module (**detect** the path: `src/store/hooks.ts`, `src/core/store/typed-hooks.ts`, …). Components import only these; raw `useDispatch` / `useSelector` are lint-banned (§7). |
| MUST | Keep RTK's dev-mode `serializableCheck` and `immutableCheck` enabled. Fix violations; never store Dates, class instances, functions, or Promises in state. `serializableCheck.ignoredActions` lists only redux-persist's lifecycle actions. The dev-only "ImmutableStateInvariantMiddleware took Nms" burst at cold start of a large persisted app is expected noise (`immutableCheck` has no `ignoredActions`); document it so nobody chases it. |
| MUST | Redux DevTools enabled only when `__DEV__` / `NODE_ENV !== "production"`. **[RN]** Hermes cannot reach the browser DevTools extension; use the Expo dev-plugin enhancer (`redux-devtools-expo-dev-plugin`) or Reactotron, attached in `__DEV__` only, with `devTools: false` on `configureStore` to avoid a dead built-in. |
| SHOULD | Provide a `setupStore(preloadedState?)` factory so tests and Storybook can create isolated stores. |
| SHOULD | **Folder layout — detect** (`rn-architecture-standards.md` §1): `features/<name>/{slice,sagas,selectors,api,types}.ts`, or a flat feature module `src/<feature>/store/{feature}-slice.ts` + `{feature}-saga.ts` (+ `{feature}-persisted-slice.ts` / `-persisted-saga.ts` when something must survive restarts). Avoid global `actions/`, `reducers/`, `sagas/` folders. One main slice per feature; a second slice only for the persisted part, never a `-ui-slice` / `-data-slice` split. |

### 2. Slices, actions, state shape

| Tier | Rule |
|---|---|
| MUST | Reducers are pure. No `Date.now()`, `Math.random()`, `crypto.randomUUID()`, or I/O inside reducers. Generate IDs/timestamps in a `prepare` callback or in the saga. |
| MUST | Collections the **store owns** (looked up by id, updated individually) are normalized with `createEntityAdapter` (`ids` + `entities`), not arrays of objects. **[C]** When a local database is the source of truth, the store holds query results (pages, ids, the current screen's rows) and the user's pending edits; it does not mirror whole tables, so the adapter rule applies only to client-created collections that are keyed by id. |
| MUST | Request status is tracked **per operation** (`idle \| loading \| succeeded \| failed`, or a project `RequestState` enum with the same four values) — e.g. `status.fetchList`, `requestStates.syncWithBackend` — never a single global `isLoading`. |
| MUST | Errors are typed (`AppError` union / discriminated enum with a `kind`), not raw strings. User-facing messages are mapped from the error kind in the UI layer (or a toast saga, §7). Existing projects that store `error: string` keep it as tracked debt and map new errors through the helper. |
| MUST | Action payloads are **objects**, never bare scalars: `PayloadAction<{ shopId: number }>`. Adding a field later is then non-breaking, and call sites read as prose. |
| SHOULD | Distinguish **server state** (fetched/synced, cached) from **UI state** (form input, selection, open modals). Keep ephemeral UI state local (`useState`) unless it must survive navigation or be tested through the store. |
| SHOULD | Separate "intent" actions from "lifecycle" actions: `loadTodosRequested` (UI intent, saga listens) vs `loadTodosSucceeded` / `loadTodosFailed` (dispatched only by sagas). A saga-only trigger may be an **empty reducer** (`checkAndSync() {}`) so the action exists for `takeEvery` without touching state. Document which actions components may dispatch. |
| SHOULD | Pick one success/failure suffix pair (`Success` / `Error`) and keep it. Mixed `Failed` / `Error` / `Rejected` suffixes become real bugs when actions are registered by name elsewhere (§9 profile C). |
| SHOULD | Mutations that hit the server are **optimistic with rollback**: apply locally, send request, revert on failure. Client-generated temporary IDs are reconciled with server IDs on success. **[C]** Not applicable: the local copy is the truth until upload, and the server's version returns through the next sync. |

### 3. Selectors

| Tier | Rule |
|---|---|
| MUST | Any selector that returns a new object/array (filter, map, sort, derived data) uses `createSelector` (`createSelector.withTypes<RootState>()`). Unmemoized derived selectors re-render on every store change. |
| MUST | Selectors live next to their slice (`selectors.ts` per feature, or exported from the slice file) and are the only way components read state; no inline `state => state.a.b.c` traversals in components beyond trivial field reads. |
| MUST | One `useAppSelector` per value. A selector that builds a new object for a component (`({ a: s.x.a, b: s.y.b })`) re-renders on every action; select `a` and `b` separately. |
| MUST | **Parameterized selectors use reselect 5's per-argument cache**: `createSelector([selectItems, (_s, id: number) => id], (items, id) => …)`, called as `useAppSelector((s) => selectItemById(s, id))` and `yield select(selectItemById, id)`. No selector factories (`(id) => createSelector(...)`), no module-level caches, no `useMemo` around the call. Reselect forwards extra arguments to input selectors, so a chained selector needs no param input of its own. |
| SHOULD | Use the entity adapter's generated selectors (`selectAll`, `selectById`, `selectIds`) rather than hand-written equivalents. |

### 4. Sagas

| Tier | Rule |
|---|---|
| MUST | Every feature saga runs through a **restart-on-crash wrapper** (`spawn` + `while(true) { try { yield call(saga) } catch (e) { report(e) } }`) so one uncaught error does not silently kill the saga tree. **Detect**: one `rootSaga` composing feature sagas, or one `sagaMiddleware.run(featureSaga)` per feature in the store module; either way, the wrapper is the rule, and a project without it tracks "a saga that throws outside its try/catch dies silently" as a known risk. |
| MUST | Every watcher uses a **deliberate** effect: `takeLatest` (cancel previous — search, filter, sort, focus-driven re-checks), `takeLeading` (ignore while running — submit buttons, upload coordinators, double-tap protection), `takeEvery` (independent fire-and-forget: loads, per-item work, fan-out triggers), `debounce` / `throttle` (rate-limited input). Mutations default to `takeLeading`; a project that uses `takeEvery` on uploads documents why it is safe (the UI is locked while the burst runs, the server tolerates duplicates) and keeps `takeLeading` as the precedent for stricter cases. `takeLatest` on a mutation is usually a bug. Watchers reference `actions.foo.type`, never string literals. |
| MUST | Network calls go through a single typed `callApi` / API client helper (base URL, auth headers, timeout, JSON decoding, error normalization). No raw `fetch` / `axios` in sagas or components. **Detect**: an axios instance with request/response **interceptors** (auth header, sync-response unwrapping, request/response logging in dev, failed-request reporting to the crash reporter) is an accepted form of this client; feature `*-api.ts` files then only build typed calls on it. |
| MUST | Long-running or screen-bound sagas are **cancelled** when no longer needed; use `race` for timeouts and `finally { if (yield cancelled()) { cleanup } }` for cleanup. |
| MUST | Every `call`/`put`/`select`/`take` is preceded by `yield`. Enforce with `eslint-plugin-redux-saga` (`redux-saga/yield-effects`). A missing `yield` fails silently. |
| MUST | **Typed yields**: `const data: SagaReturnType<typeof selectData> = yield select(selectData);` and `const res: SagaReturnType<typeof api.post> = yield call(api.post, …)` (`SagaReturnType` unwraps promises). Mandatory in new saga code, applied to old code when trivial while touching it. Add null checks for required selected data and `throw` with a message, not a silent return. |
| MUST | Sagas never swallow errors: `catch` → `put(failedAction(normalizedError))` + report to crash tooling + `console.error`. Empty `catch {}` is forbidden. The failed action also clears the loading state so the UI leaves its spinner. |
| MUST | **[RN]** A saga that must be correct after an app restart never reads a snapshot flag that was computed by an entry action (§6 navigation persistence rule); it calls a live, scoped read (DB query or selector over persisted data) instead. |
| SHOULD | Keep sagas for **orchestration**: auth/token refresh with request queueing, multi-step flows, retries with backoff, sockets, polling, offline sync queues, DB-backed list queries (profile C). Plain CRUD that is request → store → render is a candidate for RTK Query (see §9, profiles A/B only). |
| SHOULD | Avoid `select` of large sub-trees inside loops; select the minimum and pass as arguments. Independent work runs in `yield all([...])`. |
| SHOULD | Use `channel` / `eventChannel` for external event sources (sockets, native modules, AppState) instead of dispatching from outside the saga world. |
| SHOULD | **Saga-level mocking for backends that are not ready**: a dev-settings flag per endpoint (`is<Feature>MockResponseEnabled`, optional outcome enum), checked first in the saga; when on, `delay` then `put` the **same success action** the real path uses with typed mock data from `mock-data/`. Downstream code cannot tell the difference, and the toggle lives on a debug screen (`rn-architecture-standards.md` §12). |

### 5. Persistence (redux-persist or equivalent)

| Tier | Rule |
|---|---|
| MUST | Explicit `whitelist` per slice — never persist request status, errors, or transient UI state. **The whitelist string must equal the reducer key in `combineReducers`**: a misspelled key means the slice is silently not persisted. A repo-invariant check or a typed whitelist (`satisfies (keyof RootState)[]`) catches it. |
| MUST | Persisted state has a `version` and a `migrate` map (`createMigrate`); bumping the shape without a migration is forbidden. Migrations are keyed by the next integer, and **each migration must accept the output of the one before it**, not today's shape: a device that skipped versions runs them all in order on its next start. |
| MUST | **When a migration is needed**: a stored field is renamed, removed, changes type, gets a new field *inside a nested object or array item*, or changes its **default** (the stored value wins over `initialState`, so a changed default never reaches existing installs without a migration). Removing a field: destructure it out with a rest spread. Purely added **top-level** fields need no migration when the refresh pattern below is in place. |
| MUST | **Reset and refresh on every persisted slice.** `reset: () => initialState` runs on the app-wide wipe action (reset app, change environment, different-user login); `refresh: (state) => ({ ...initialState, ...state })` runs on **every app start** so newly added top-level fields exist on old installs. Every whitelisted slice listens to both; a whitelisted slice without the reset handler silently survives "reset app". Slices whose data belongs to the environment (selected backend, tenant) honour a `shouldSkipEnvironmentReset` payload flag. |
| MUST | **Transient state kept in a persisted slice is reset explicitly in `refresh`** (`modalOpen: false`, `productOrderHistory: initial…`), otherwise a killed app restores it and a modal reopens on the next launch. |
| MUST | Secrets (tokens, passwords) are never in redux-persist; they live in secure storage (`rn-runtime-quality-standards.md` §A). When a persisted slice still holds something sensitive, the crash reporter's state transformer redacts it. |
| SHOULD | Separate persisted slices from non-persisted ones at the reducer level (`{feature}-persisted-slice.ts`, or nested `persistReducer` per slice) rather than persisting the root. |
| SHOULD | **"Once per user per install" = a persisted flag**: store `hasInitializedX` in a persisted slice and gate the one-time saga on it; a normal logout keeps it, the wipe paths clear it. |
| SHOULD | A debug screen (or a dev command) exposes `resetAllAppState` / `refreshAllAppState` and the persisted store's raw contents (AsyncStorage key `persist:root`), so a persisted-shape bug can be reproduced without reinstalling. |

### 6. React Native specifics **[RN]**

| Tier | Rule |
|---|---|
| MUST [RN] | Screen-bound sagas start on screen **focus** and cancel on **blur** (React Navigation `useFocusEffect`), not on mount/unmount — screens stay mounted in the stack. |
| MUST [RN] | `AppState` changes (`active` / `background`) are dispatched as actions so sagas can pause polling/sync and resume on foreground. Long forms save their draft to storage on background and restore on foreground (`rn-architecture-standards.md` §7). |
| MUST [RN] | Auth token refresh is a saga that **queues** concurrent requests while a refresh is in flight and replays them afterward (single-flight refresh). Projects without refresh tokens (session token + re-login on 401) document that and route every 401 through one interceptor that shows the session-expired flow — except for the logout request itself. |
| MUST [RN] | **Navigation state persistence changes what re-runs.** When the navigator restores a deep screen after an app kill, the entry action that normally precedes it (visit start, flow start) does **not** fire again. Any non-persisted slice value derived only from that entry action is stale after the restart: hidden buttons, filtered lists, missing records. Rules: sagas read live scoped data (§4); UI flags in non-persisted slices are recomputed in the feature saga's `refreshAllAppState` handler when the persisted state says the flow is open, **and** on focus of the flow's home screen (`takeLatest`, so a slower in-flight check cannot overwrite a newer result). |
| SHOULD [RN] | Lists render through FlashList with entity ids as data and a memoized row component selecting its own entity by id (`selectById`), so one row update does not re-render the list. **[C]** Large lists (1,000+ rows) are filtered, sorted and paged in SQL, and the slice holds the current page (see §9 profile C). |
| SHOULD [RN] | Offline-first apps: an explicit **outbox/sync queue** slice (pending mutations with status, retry count, last error) drained by a saga that reacts to connectivity (`@react-native-community/netinfo` via `eventChannel`) **or** to user-driven sync points (§9 profile C). |
| SHOULD [RN] | Hermes-safe state: no class instances, Maps/Sets, or functions in state (also required by `serializableCheck`). |

### 7. Tooling & observability

| Tier | Rule |
|---|---|
| MUST | ESLint: `no-restricted-imports` banning raw `useDispatch` / `useSelector` from `react-redux` (use the typed hooks), and `eslint-plugin-redux-saga` rules enabled. The saga/slice file glob may relax `no-unused-vars` / `no-empty-function` (unused `action` parameters, empty trigger reducers). |
| MUST | A logging middleware records **action types** (not payloads — PII) as breadcrumbs for Sentry / Crashlytics / equivalent; the crash reporter's redux enhancer attaches state with a **state transformer that redacts** tokens and passwords (`rn-runtime-quality-standards.md` §B). |
| MUST | **Toasts for errors live in one saga**, not in components: `takeEvery(actions.fooError.type, showToast, { key })` per action, text as an i18n key. Components dispatch intents and render state; they do not decide what to toast. |
| SHOULD | On-device inspection **[RN]**: the Expo DevTools redux plugin or Reactotron; a dev-only `redux-logger` (collapsed, diff) through a logger that truncates large arrays. Hidden action families (`^coreData`, `^alert/`) are listed in the docs. |
| SHOULD | **A CLI that reads the live state of the running dev app** (`npm run redux-state -- --path=feature.field`, `--actions=30`, `--out=file`) over the dev-plugin relay, so a developer or an agent can verify a saga's effect without a browser tab. Document its limits: arrays truncated, relay can stall or go stale (fall back to logcat / the persisted store on disk), and a relay `--dispatch` reaches **reducers only — sagas never fire for it**, so inject state with it and trigger sagas through the UI. |
| SHOULD | Tests (when the project is at testing Level ≥ 1, `rn-testing-standards.md` §0): slice reducers tested as pure functions; sagas tested with `redux-saga-test-plan` (`expectSaga` for integration-style, `testSaga` for step-by-step; note the package is unmaintained since 2022 but works with redux-saga 1.x); selectors tested for memoization where it matters. At Level 0: a debug screen per feature and the live-state CLI are the verification tools, and the docs say so. |

### 8. API & data layer

| Tier | Rule |
|---|---|
| MUST | One `services/api/` client with a typed interface (`ApiClient`), injected into sagas via saga context (`getContext('api')`) or a dependency module, so tests swap in a fake without `jest.mock`. Features add typed methods (`api.todos.list()`), never build URLs ad hoc. **Detect**: an axios instance + per-feature `*-api.ts` modules that take the base URL from a selector counts, provided no feature constructs its own instance. |
| MUST | The client owns: base URL (env model A: from env; model B: from the environment selector slice), auth header injection, timeout (`AbortController`), JSON decoding **validated with zod** at the boundary, and normalization of every failure into `AppError` (`kind: 'network' \| 'timeout' \| 'auth' \| 'forbidden' \| 'notFound' \| 'validation' \| 'server' \| 'unknown'`, with `status`, `retryable`, and a `cause`). Nothing above the client sees raw `Response`/`AxiosError`. |
| MUST | **Validation mode is a decision**: *strict* (parse failure = `validation` error, data rejected) or *report-only* (safe-parse, failure reported to the crash reporter with the zod issues, data still used). Report-only fits backends that are the source of truth and evolve ahead of the app; strict fits payloads the app cannot safely use half-valid. Record the mode per endpoint family; a shared `safeParseAndReport(schema, data)` helper implements report-only in one place. |
| MUST | Auth: 401 triggers the single-flight refresh saga; concurrent requests queue and replay; refresh failure logs out. Access token in memory, refresh token in secure storage (see `rn-runtime-quality-standards.md` §A). |
| MUST | Retries only for `retryable` errors (network, 502/503/504, timeout), with exponential backoff + jitter and a max attempt count, implemented once in a `retry` saga helper — never ad-hoc loops. Non-idempotent requests (POST without idempotency key) are not retried. |
| MUST | Every request carries a correlation/request id header (`X-Request-Id`, uuid) logged as a Sentry breadcrumb so client errors can be matched to server logs. A backend version response header, when the backend sends one, is read in one interceptor into a slice and attached to crash reports as a tag. |
| MUST | Offline-first apps: an **outbox** (pending mutations persisted per feature, with attempts and last error) drained by a saga on connectivity / app foreground / user-driven sync points, with per-item idempotency keys where the server supports them; conflict policy (last-write-wins, server-wins, merge) documented per entity. Profile C refines this in §9. |
| SHOULD | Pagination handled uniformly: cursor-based preferred; the slice stores `ids` per page plus `nextCursor`/`hasMore`; `takeLeading` for load-more. **[C]** SQL `LIMIT/OFFSET` paging with `offset = currentRows.length`, `totalCount` from `COUNT(*)`, `hasMore` derived. |
| SHOULD | Caching policy explicit per entity: TTL in state (`fetchedAt`) and a `staleWhileRevalidate` helper saga, or RTK Query where the project profile allows (§9). |
| SHOULD | Types generated from the OpenAPI spec when one exists (`openapi-typescript`), with zod schemas derived or hand-written for the subset actually used; drift checked in CI. |
| SHOULD | Request/response logging in dev only, through the logger with redaction; never in production builds. Interceptors that would register in production stay behind `if (!__DEV__) return`. |
| SHOULD | Mocking for development: the same `ApiClient` interface with an in-memory implementation (`createFakeApi`) selectable via a dev flag — doubles as the test fake. At Level 0, the saga-level mock branch (§4) plays this role. |

### 9. Project profile: saga-centric, CRUD-centric, or DB-centric

Three legitimate project profiles. The audit prompt detects the profile and asks before recommending anything that assumes another one.

- **A. Saga-centric** — online apps with complex orchestration, sockets, local-first data with sync events. Sagas own all async; the store is the source of truth. RTK Query is *not* recommended; its cache model fights an outbox.
- **B. CRUD-centric** — online apps where most features are fetch → show → mutate → refetch. RTK Query should handle data fetching/caching; sagas remain only for orchestration (auth, flows, sockets).
- **C. DB-centric offline-first** — field apps that must work for days without a connection. A **local database** (SQLite) is the source of truth for reference data synced from the backend; the store holds query results and the user's pending work; sagas own sync and queries. RTK Query is not applicable. The rules below are **MUST [C]** unless marked otherwise.

Mixed projects are fine: pick per feature, document the rule in the project docs.

#### Profile C rules

| Tier | Rule |
|---|---|
| MUST [C] | **Where data lives is a decision tree in the docs.** Comes from the backend sync → the synced DB. Created by the user on the device → a persisted slice (outbox), **not** the DB. Device telemetry that is uploaded and dropped → its own small DB or table. Client-only metadata → a separate client DB. One table per concern; never mix client writes into synced tables. |
| MUST [C] | **The synced database is client-read-only.** Its only writers are the sync pipeline (full sync, delta sync, paged sync of large tables) and dev-only mock tooling. Client-created records (orders, visits, filled forms, notes, pictures) wait in their persisted slice until the upload succeeds, are **dropped on success**, and appear in the DB only when the backend sends them back in a later sync. A derived table rebuilt from synced tables (search index) is the one accepted client-built table. Never add client INSERT/UPDATE paths into synced tables. |
| MUST [C] | **Backend-driven by design.** The app keeps no record of what it uploaded. If uploaded data does not come back, or data that should be invalid stays valid after a sync, the first check is **what the sync delivered**; a missing update there is a backend issue. Do not add client-side duplicate guards, an "already uploaded" memory, or upload-response parsing to paper over it. The agent-instructions file states this, because every assistant's reflex is to add the guard. |
| MUST [C] | **Push-then-pull.** After a batch of uploads, the app pulls so on-device data reflects what the server now holds. The hazard is pulling **too early**: a pull that runs while an upload is in flight returns a response without it, and the local copy is already dropped on success. One non-persisted slice holds the set of in-flight uploads; every upload action is **registered with all three of start / success / error** (start adds its key, both terminal actions remove it); the pull fires when the set empties. Missing start → early pull; missing success or error → the set never empties, the pull never fires, and a UI that locks during sync stays locked until restart. Per-item uploads register with an item suffix. The registration is a checklist step in "how to add an upload", with the exact action names, because a name mismatch leaks a key forever. |
| MUST [C] | **An upload queue is a slice with a trigger and a triplet**: `checkAndSync` (empty reducer, saga reads the queue and returns if empty — then no upload action, no pull), `sync` (status pending), `syncSuccess` (status resolved, queue emptied), `syncError` (status rejected, queue kept). The app-wide "save everything" saga `put`s every feature's check action; each feature uploads independently; the shared in-flight set decides when the pull starts. An empty burst pulls nothing. |
| MUST [C] | **When syncs run is an invariant, written down.** Typical: user-driven only (login, focus of the main list screens, manual sync, flow exit), never periodic, and **never during a work session** (a shop visit, a job) — so the synced DB is immutable for the whole session and in-session code may rely on it. In-session network calls that do not touch the synced DB are listed explicitly. |
| MUST [C] | **Full sync vs delta sync.** The first sync (or after a wipe) fetches the full state; later syncs send the last sync timestamp and receive deltas plus `deletedIds`. The timestamp advances only when **every** table stored successfully; one failed table keeps it, so the next delta covers the same window. Nothing truncates a synced table; rows disappear only via `deletedIds` or a wipe. Tables over a size limit page in after the full sync; code that reads such a table waits for the paged sync's completion action, not for the HTTP success. |
| MUST [C] | **Wipe paths** (reset app, change environment, different-user login) all run the same steps in order: reset the sync timestamp **before** the wipe (a kill mid-wipe must still lead to a full sync, not a delta on empty tables), drop and recreate the local databases, then dispatch the app-wide `reset` to every persisted slice. A normal logout wipes nothing: the same user's pending work survives. A signing-key change forces a reinstall, which deletes everything unsynced (`rn-cicd-standards.md` §3b). |
| MUST [C] | **UI locks during sync.** One selector (`selectIsSyncInProgress`) is true from the config fetch through the upload window, the pull, the DB write and the paged sync; it disables navigation into flows that create data. This is what makes `takeEvery` uploads and `takeLeading` coordinators safe: the queue a burst snapshots cannot grow underneath it. |
| MUST [C] | **Batch uploads validate before queueing.** When one request carries the whole queue, one invalid item (a `null` the server rejects) fails the batch, the queue stays, and every later trigger fails again. Validate required fields at enqueue time, and give the user a way to remove a bad queued item. |
| MUST [C] | **Database discipline:** parameterized values only (table and column names from an enum, never from user or backend strings); SQL-first filtering, sorting, search and paging for lists over ~1,000 rows (`COLLATE NOCASE` in SQL, not `Intl.Collator` in JS; merge local unsaved edits into each page after the query); batch inserts split under the engine's bound-variable limit (SQLite: 32,766); a denormalized search-index table for complex text search; no N+1 query loops. Single-row reads follow the `noUncheckedIndexedAccess` contract (`rn-project-standards.md` §3). |
| MUST [C] | **Schema versioning per database:** a `dbVersion` table, idempotent migrations keyed by version, **and** the `CREATE TABLE` statements extended for fresh installs. The DDL reference files in the repo are generated by a script from a real device DB and never hand-edited. After a schema change the dev app needs a **cold restart** (terminate + launch): Fast Refresh runs the new queries before DB init re-runs, and the "no such column" it produces is not a bug. |
| MUST [C] | **Adding a synced table is a numbered process** in the docs (DTO + zod schema, insert config with nullability, table enum, create/migrate, per-table update saga registered with the timestamp watcher, paging eligibility, mock data on a debug screen, DDL regeneration). Skipping the timestamp watcher is the classic failure: the table stores, the timestamp never advances. |
| SHOULD [C] | Dev tooling: scripts that extract the databases from the simulator/emulator and regenerate the DDL files; a debug screen per synced feature with insert-mock / clear / count actions (the one accepted client write into the synced DB, dev-only, always paired with the clear action); a sync debug screen showing the in-flight set and the last timestamp. |
| SHOULD [C] | Known limitations are listed, not hidden: duplicate POSTs under rapid re-trigger (server tolerates), app kill mid-upload (in-flight set is not persisted; the queue is, so the next burst re-uploads), one failing table blocking the timestamp. |

Note for the audit: §8 (API & data layer) and §9 profile C are audited by this prompt; the prompt's Phase 2 sizing applies — an `ApiClient` + `AppError` introduction is typically M, migrating all sagas onto it L, an outbox for an offline app L–XL, the push-then-pull registry for an existing offline app M (the hard part is the checklist, not the code).

---

## Part 2 — Audit prompt (for Claude Code / agentic tools)

Paste everything between the markers. Keep Part 1 of this file in the repo (or attach it) — the prompt refers to it as **"the rules"**.

```text
<<<<< PROMPT START >>>>>

You are auditing this repository's Redux + redux-saga code against a set of best practices ("the rules"). The rules are in the file `redux-saga-best-practices.md` (Part 1) — read it first. If the file is not in the repo, ask me to provide it before doing anything else.

Work in the phases below, in order. Do not modify any source code (outside of documentation and the migration plan file) until I explicitly approve in Phase 6.

## Phase 0 — Orient
1. Read package.json: confirm react-redux, @reduxjs/toolkit (or plain redux), redux-saga, redux-persist, react-navigation, netinfo, RTK Query usage, a local database (expo-sqlite, op-sqlite, WatermelonDB, realm), eslint plugins, test libraries, react-redux version (affects `withTypes` availability).
2. Determine platform: React Native, React web, or both (monorepo). Apply [RN] rules only to RN code.
3. Locate: store setup (root saga or per-saga `run`, DevTools approach, serializable/immutable check config), all slices/reducers, all sagas, selectors, API client(s) and interceptors, persistence config (whitelist, version, migrations, reset/refresh handlers), lint config, existing tests or the declared verification level (`rn-testing-standards.md` §0), debug screens and mock-data folders, live-state / DB extraction scripts.
4. Count: watchers by effect (`takeEvery` / `takeLatest` / `takeLeading` / `debounce`), sagas with and without try/catch, `yield select` with and without `SagaReturnType`, selector factories, `useSelector` calls that build objects, persisted slices with and without reset/refresh handlers, whitelist keys that match no reducer key.
5. Detect existing documentation and conventions: README, docs/ or another docs tree, ADRs, CONTRIBUTING, CLAUDE.md, AGENTS.md, .cursorrules, .github/copilot-instructions.md, or similar. Note their style and where architecture rules currently live. You will extend these rather than invent a parallel system.

## Phase 1 — Detect project profile and ask
Classify the app as A saga-centric, B CRUD-centric, C DB-centric offline-first, or mixed, with evidence: count of sagas that are plain request→put vs orchestration; presence of a local database and which tables the client writes; outbox/sync queue slices; sync triggers (periodic, connectivity, user-driven); sockets; offline handling.
Then STOP and ask me:
- "Is this classification right?"
- For A/B: "Should I evaluate the RTK Query recommendation (rules §9) for this project: yes / no / only flag it as optional?"
- For C: "Confirm the sync timing invariant I found (when syncs run, whether a work session freezes the synced DB) and the validation mode (strict / report-only) per endpoint family."
- The typed-hooks path and folder layout option to keep.
Wait for my answer before continuing.

## Phase 2 — Audit
For every rule in Part 1 (keep the section numbering; include §9 profile C rules only for profile C), record:
- Status: ✅ compliant / ⚠️ partial / ❌ missing / ➖ not applicable (with reason)
- Evidence: file paths and symbol names (not line numbers)
- Tier: MUST / SHOULD
- Fix size: S / M / L / XL, plus an approximate count of files touched
- Risk: low / medium / high (behavioral change, persisted-state migration, auth flows, sync orchestration are high)

Sizing guide: S = under 1h, a few files, mechanical; M = half a day, a feature's worth of files; L = a few days, cross-cutting; XL = a week+, touches most features or persisted state.

Do NOT fabricate compliance. If you cannot find evidence, mark it ❌ or ⚠️ and say what you looked for.

## Phase 3 — Gap report (in chat)
Present:
1. A summary table: rule → tier → status → size → files → risk, grouped MUST first, then SHOULD.
2. Totals: number of MUST gaps, SHOULD gaps; total effort as T-shirt size and total files touched for "all MUST", "all MUST + SHOULD".
3. Top 5 highest-value fixes (value = user-facing risk removed ÷ effort), with one sentence of justification each. For profile C, typically: upload registration audit (every upload has start/success/error registered), persisted-slice reset/refresh coverage, whitelist-key check, sync timing invariant written down, validation mode decision.
4. Anything in the codebase that intentionally deviates from a rule for a good reason — propose documenting it as an accepted exception rather than fixing it.

## Phase 4 — Documentation (write now, no source changes)
Make future code follow the rules:
1. If an agent-instructions file exists (CLAUDE.md, AGENTS.md, .cursorrules, copilot-instructions), add or extend a concise "Redux & saga conventions" section: the MUST rules as short imperative bullets, project-specific decisions (profile, folder layout, typed hooks path, API client path, validation mode, accepted exceptions), and a pointer to the full doc. For profile C add the three invariants that assistants break most: synced DB is client-read-only, no client-side guards against backend-driven data, no sync during a work session. Keep it short; agents read this on every task.
2. If a docs tree or architecture doc exists, add or extend a full conventions document there (all MUST and SHOULD rules, adapted to this project's actual paths and names, with one short code example per section taken from or adapted to this codebase). For profile C also a sync-orchestration doc (push-then-pull diagram, registration contract, "how to add an upload" checklist, debugging section) and a database doc (decision tree, wipe paths, query rules). If no docs location exists, create `docs/redux-saga-conventions.md` and link it from README.
3. Match the existing docs' tone, heading style, and language.
4. Do not duplicate: if a rule is already documented, reference it instead of restating.
Show me a diff summary of the documentation changes.

## Phase 5 — Migration plan file
Create or update `redux-saga-migration-plan.md` in the docs location (use an existing plans/ADR location if the project has one) containing:
- Date, project profile decision from Phase 1
- The full gap table from Phase 3 with a checkbox per item: `- [ ] §4.1 Restart-on-crash wrapper — M, 3 files, low risk`
- Suggested order: MUST + low-risk first, then MUST + high-risk, then SHOULD by value
- An "Accepted exceptions" section
You will update the checkboxes as items are completed in Phase 6.

## Phase 6 — Ask before applying
Ask me, with a clear recommendation:
- "Apply everything?" — state total size and risk
- "Apply all MUST only?" — state size and risk
- "Pick items" — list them by §number so I can reply with a subset
- Your recommendation and why (default recommendation: all MUST items that are low/medium risk now; high-risk MUST items (persistence migrations, auth refresh, sync registry changes) as separate reviewed PRs; SHOULD items opportunistically when touching the feature).
Wait for my answer. Do not start refactoring on an ambiguous reply — ask again.

## Phase 7 — Apply (only what I approved)
For each approved item:
1. One logical change at a time; keep commits/PR-sized units per §item or per feature.
2. Preserve behavior unless the rule explicitly requires a behavior change (e.g. takeLatest → takeLeading on a mutation); call those out explicitly.
3. Run typecheck and lint after each item; run existing tests when a runner exists. If the project is at testing Level ≥ 1 and tests do not exist for the touched saga/slice, add the minimal test that proves the change (reducer pure test, or redux-saga-test-plan `expectSaga`). At Level 0, describe the manual verification (which debug screen, which live-state path to read, which device steps) and run what you can on the simulator.
4. Tick the item in the migration plan file and record what changed.
5. If an item turns out larger or riskier than estimated, stop, explain, and ask before continuing.
At the end, summarize what was done, what remains, and update the migration plan file's totals.

General constraints:
- Never store or log payloads that may contain PII when adding logging middleware.
- Never change persisted-state shape without a `migrate` entry and version bump; never change a persisted default without one either.
- Profile C: never add a client write path into a synced table, a client-side guard against backend-owned data, or a sync trigger inside a work session.
- Prefer extending existing helpers (API client, hooks) over creating duplicates.
- If the repo uses plain Redux without RTK, treat "migrate to configureStore + createSlice" as the first MUST item and size it separately.

<<<<< PROMPT END >>>>>
```

### Adapting the prompt

- **Chat-only LLM (no repo access):** replace Phase 0 with "I will paste store.ts, rootSaga, one representative slice + saga, lint config and package.json; ask for anything else you need," and replace Phase 4/5/7 writes with "output the file contents for me to add."
- **Web-only project:** tell it up front "React web, skip all [RN] rules" to save a round-trip.
- **Online-only project:** tell it "profile A or B, skip §9 profile C" to save a round-trip.
- **Re-running later:** point it at the existing `redux-saga-migration-plan.md` and ask it to re-audit and update statuses only.
