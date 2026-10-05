# React Native Testing & Verification Standards + Audit Prompt

> Standards version: 2 (2026-10-05). What changed: see `STANDARDS-INDEX.md` → Version history.

Part 1: rules (MUST / SHOULD). Part 2: audit prompt for Claude Code. Companion to the other `*-standards.md` files.

Assumes: Jest, React Native Testing Library (RNTL), redux-saga-test-plan, **Maestro** for E2E (recommended; existing Detox is kept if present), MSW optional (not standard — API mocking is done at the saga/API-client boundary). A project **without** a test runner is handled by §0: it declares a verification level, and the rules of the higher levels apply only from the level it declares.

---

## Part 1 — Rules

### 0. Verification levels — declare one

Not every project has automated tests, and pretending otherwise is worse than saying so: an agent that assumes `npm test` exists writes tests nobody runs, and a reviewer that assumes coverage skips the manual pass. The project **declares its level** in the docs and the agent-instructions file, and the rules below are read from that level.

| Level | What exists | Who it fits |
|---|---|---|
| **0 — Verified without a runner** | Static checks + repo-invariant scripts + debug tooling + device-driven verification (below). No Jest, no `test` script, and the docs say so plainly. | Small teams on a large legacy app where the test infrastructure debt is bigger than the feature backlog; apps whose risk sits in sync/DB flows that unit tests do not reach. |
| **1 — Unit + saga** | Jest with reducers, selectors, utils, zod schemas, saga tests. Everything from Level 0 stays. | The first step out of Level 0; most value per hour. |
| **2 — + component & integration** | RNTL screen tests (four states), navigation-level tests, `renderWithProviders`, fake API client. | Apps with a stable primitive layer and a store factory. |
| **3 — + E2E** | Maestro smoke flows on built artifacts in CI. | Apps with a release pipeline (`rn-cicd-standards.md` profile A). |

**Level 0 rules** (all MUST; they remain MUST at every higher level):

| Tier | Rule |
|---|---|
| MUST | **The docs state the level.** The testing doc opens with "There are no automated tests" (or the level), what verification means instead, and what would trigger moving up. The agent-instructions file repeats it in one line so no assistant invents a runner. |
| MUST | **Static checks are green before any work is handed back**: `typecheck`, `lint` (max warnings 0), `format:check`, plus the repo-invariant scripts in pre-commit (`rn-project-standards.md` §6). These are the only automated gate, so they are never skipped or weakened. |
| MUST | **`testID` on every interactive element** (`rn-architecture-standards.md` §2), with the stable naming convention, in new code and in any touched code. It is the hook for device-driven verification now and for Maestro later; a project that reaches Level 3 must not start by retrofitting ids. |
| MUST | **Debug tooling per feature** (`rn-architecture-standards.md` §12): a debug screen with mock data insert/clear, saga-level mocks for endpoints that are not ready, flag overrides. A change that cannot be exercised through the UI without the real backend is not verifiable, and the task asks for the tooling. |
| MUST | **Device-driven verification is scripted and documented**: a guide (with a "verified on" date and toolchain versions) for screenshots, taps, swipes, typing and accessibility-tree dumps on the iOS simulator and the Android emulator, driven from the terminal (e.g. `xcrun simctl` + an accessibility CLI on iOS, `adb` + `uiautomator dump` on Android), including the coordinate systems (points vs pixels), how to tap by `testID`, how to reload / cold-restart the app, and the known quirks of the current toolchain. An agent follows it to verify its own changes and reports what it verified and what it could not. |
| MUST | **Live state is inspectable from the terminal**: a CLI for the running dev app's store (`redux-saga-best-practices.md` §7), scripts that extract the local databases from the simulator/emulator, the persisted store's on-disk location, and logcat as the authoritative fallback. The debugging workflow is **debug-first**: read the code path, look at what the app actually holds, add prefixed temporary logs, reproduce, then fix — never fix from the symptom. |
| MUST | **Every change reports its verification**: what was run (static checks with real output), what was checked on which platform, what was *not* verified and what the reviewer must check by hand. A manual verification checklist with the project's recurring items (offline pass, restart pass, both platforms, large-data pass, translations, new table on a fresh install) lives in the docs and is referenced, not retyped. |
| MUST | **No direct backend calls while investigating.** The agent observes what the app received (live state, persisted store, extracted DB, logs); it does not `curl` the backend with the app's token unless the owner asks. A field the app does not yet store is invisible in state, and the agent says so instead of probing. |
| SHOULD | The trigger for Level 1 is written down (e.g. "when the first shared saga helper is refactored", "when the primitive layer is stable"), together with the first candidates (sync orchestration, auth, persisted-state migrations). |
| SHOULD | Flaky behaviours found during device verification (tap style that reports success but does nothing, reload that is a no-op after a websocket loss) are recorded in the simulator guide with the workaround, dated. |

### 1. The pyramid and what each layer owns (Level ≥ 1)

| Layer | Tool | Tests | Doesn't test |
|---|---|---|---|
| Unit | Jest | reducers, selectors, utils, zod schemas, error mapping, pure hooks logic | rendering, navigation |
| Saga | redux-saga-test-plan | orchestration: order of effects, cancellation, error paths, token refresh, push-then-pull registration | HTTP itself |
| Component | RNTL | one component/screen with a real store and mocked API client: what the user sees and does | internal state, styles, snapshot of the whole tree |
| Integration | RNTL + navigation | a flow across 2–3 screens with the real navigator | native modules |
| E2E | Maestro on a real build | the 5–10 golden paths (install → onboard → core action → logout) | edge cases, every form validation |

| Tier | Rule |
|---|---|
| MUST | Every layer up to the declared level exists in the repo (E2E may start with one flow). An empty layer below the declared level is a gap; a layer above it is not. |
| MUST | Ratio guidance: most tests are unit + saga (fast, deterministic); component tests cover each screen's happy path + error/empty states; E2E stays under ~15 flows. |
| MUST | Tests run headless in CI with no device dependency except the E2E job. |
| SHOULD | Coverage is reported but not worshipped: thresholds set at the current level and ratcheted up; 100% is not a goal. |

### 2. Jest setup

```js
// jest.config.js
module.exports = {
  preset: 'jest-expo', // or 'react-native' for bare
  setupFilesAfterEnv: ['<rootDir>/jest.setup.ts'],
  moduleNameMapper: { '^@/(.*)$': '<rootDir>/src/$1', '^@assets/(.*)$': '<rootDir>/assets/$1' }, // must match tsconfig paths
  transformIgnorePatterns: [
    'node_modules/(?!((jest-)?react-native|@react-native(-community)?|expo(nent)?|@expo(nent)?/.*|@expo-google-fonts/.*|react-navigation|@react-navigation/.*|@sentry/react-native|native-base|react-native-svg|nativewind|react-native-css-interop|@rn-primitives/.*)/)',
  ],
  collectCoverageFrom: ['src/**/*.{ts,tsx}', '!src/**/*.d.ts', '!src/**/index.ts', '!src/app/**', '!src/**/*.stories.tsx', '!src/**/*DebugScreen.tsx', '!src/**/mock-data/**'],
  coverageThreshold: { global: { branches: 40, functions: 50, lines: 50, statements: 50 } }, // ratchet up
  testPathIgnorePatterns: ['/node_modules/', '/e2e/', '/.maestro/'],
  clearMocks: true,
  testTimeout: 10000,
};
```

| Tier | Rule |
|---|---|
| MUST | `jest-expo` (Expo) or `react-native` preset; `moduleNameMapper` mirrors `tsconfig.paths`; `transformIgnorePatterns` lists every RN library that ships untranspiled code (the #1 cause of "SyntaxError: Unexpected token export"). |
| MUST | One `jest.setup.ts` with: `@testing-library/react-native` matchers, `react-native-gesture-handler/jestSetup`, Reanimated mock (`react-native-reanimated/mock` or `jest-setup`), `@react-native-async-storage/async-storage/jest/async-storage-mock`, safe-area mock, netinfo mock, the React Compiler runtime untouched (compiled components test like any other), and the global `fetch` disabled (throws if called — forces mocking at the API client). DB-centric apps mock the SQLite module at the adapter boundary, or run an in-memory SQLite for DB-layer tests. |
| MUST | `clearMocks: true`; no test relies on mock state from another test. |
| MUST | Tests colocated: `Foo.test.tsx` next to `Foo.tsx` (or `__tests__/` per feature); no global `tests/` folder for unit/component tests. |
| MUST | Scripts: `test` (watch, local), `test:ci` (`--ci --coverage --maxWorkers=50%`), `test:e2e` (Maestro). |
| SHOULD | Fake timers by default for saga/debounce tests (`jest.useFakeTimers()` in the test, not globally — RNTL and fake timers need care). |
| SHOULD | `jest-extended` only if actually used; avoid matcher-library sprawl. |

### 3. Test utilities (`src/test/`)

| Tier | Rule |
|---|---|
| MUST | `renderWithProviders(ui, { preloadedState?, store?, route? })` wraps in Redux `Provider`, `SafeAreaProvider` (with `initialMetrics`), navigation container (or Expo Router test harness), i18n, the theme/NativeWind provider and, mid-migration, the legacy UI library's provider. Every component test uses it; raw `render` is banned via `no-restricted-imports`. |
| MUST | Factories, not fixtures: `makeTodo(overrides?)`, `makeUser(overrides?)` producing valid typed objects with deterministic ids (`@faker-js/faker` seeded, or a counter). No `JSON` fixture files copied from prod. The feature's `mock-data/` (used by debug screens) may back the factories so the two stay in sync. |
| MUST | A fake API client implementing the real `ApiClient` interface (`createFakeApi({ todos: [...] })`) injected into the store's saga context; tests never mock `fetch`/`axios` directly. Projects whose client is an axios instance with interceptors fake it at the `*-api.ts` module level with `jest.mock`, consistently. |
| MUST | Test ids come from the project's `testID` convention (`scope.element`, business-key suffix; `rn-architecture-standards.md` §2). **Detect**: a typed catalog shared with Maestro (`tid('todos.addButton')`) or the plain convention with ids written at the call site and found by grep. Either way the same id strings serve RNTL, Maestro and the simulator CLI; strings are never maintained in two places. |
| SHOULD | `createTestStore(preloadedState?, { api })` = real reducers + real sagas + fake API, so component tests exercise the actual data flow. |
| SHOULD | Navigation helper for integration tests: `renderWithNavigation(<Stack />, { initialRouteName, params })`. |

### 4. Unit tests

| Tier | Rule |
|---|---|
| MUST | Reducers tested as pure functions: `reducer(state, action)` → expected state, including unknown action and initial state; persisted slices also `reset` and `refresh` (new top-level field appears, transient state cleared). |
| MUST | Selectors: tested for result **and**, for memoized ones, for referential stability on unchanged input (`expect(sel(s)).toBe(sel(s))`); parameterized selectors for stability per argument. |
| MUST | zod schemas tested with at least one valid and one invalid case per rule that matters (business rules, not zod itself). |
| MUST | Error mapping (`AppError` from HTTP status/network failure) fully covered — it's small and every screen depends on it. |
| MUST | **Persisted-state migrations** tested: for each migration, a stored state of version N−1 passes through `migrate` and yields the expected shape; the chain from the oldest supported version runs end to end. |
| SHOULD | Utilities use table tests (`it.each`) for input/output matrices; date helpers tested across a DST boundary and a UTC/local midnight. |
| SHOULD | Pure hook logic extracted to functions and tested without `renderHook` when possible; `renderHook` from RNTL for the rest. |

### 5. Saga tests — redux-saga-test-plan

| Tier | Rule |
|---|---|
| MUST | Every watcher saga has an `expectSaga` test with `.withReducer` and `.provide([...])` for `call`s to the API/services/DB, asserting the final state and the `put`s, for: success, failure, and (where relevant) cancellation. |
| MUST | Cancellation semantics are tested where they matter: `takeLatest` (second request cancels first), `takeLeading` (second ignored), `race` timeouts. |
| MUST | Token refresh / request queue saga (if present) has a test proving concurrent 401s trigger exactly one refresh and all requests replay. |
| MUST | **DB-centric profile**: the push-then-pull registry has tests proving (a) an upload's start adds its key and both terminal actions remove it, (b) the pull fires only when the set empties, (c) an empty burst fires no pull; the "add an upload" checklist includes adding the registration test. The sync timestamp test proves it advances only when every table stored. |
| SHOULD | `testSaga` (step-by-step) only for sagas where effect order is the contract; prefer `expectSaga` otherwise. `redux-saga-test-plan` has had no release since 2022; it works with redux-saga 1.x, and the docs note the maintenance status so nobody is surprised. |
| SHOULD | Providers use `matchers.call.fn(api.fetchTodos)` style dynamic providers rather than brittle `[call(api.fetchTodos), result]` tuples. |

### 6. Component & screen tests — RNTL

| Tier | Rule |
|---|---|
| MUST | Query priority: `getByRole` / `getByLabelText` (accessibility) → `getByText` → `getByTestId` last. Using testID when a role query would work is a review comment — it also enforces a11y. (The testIDs still exist for Maestro and device tooling; they are just not the first query.) |
| MUST | Interactions via `userEvent` (RNTL ≥ 12.2) with fake timers set up per the RNTL docs (`userEvent.setup({ advanceTimers: jest.advanceTimersByTime })`); `fireEvent` only for events `userEvent` doesn't support. RNTL 14 changed its renderer dependency (`test-renderer` peer, React ≥ 19, RN ≥ 0.78); pin the major that matches the project's React and check the migration notes before bumping. |
| MUST | Async UI awaited with `findBy*` / `waitFor`; no `act` warnings tolerated — CI fails on `console.error` (`jest-fail-on-console` or a setup guard). |
| MUST | Each screen has tests for: happy path, loading, empty, error (with retry). |
| MUST | Behavior, not implementation: no asserting on store action types from a component test, no `instance()` access, no checking `style` props except for a specific visual-logic rule (e.g. disabled opacity). |
| MUST | No whole-tree snapshot tests. Inline snapshots are allowed for small serialized outputs (a formatted string, a small object). |
| SHOULD | Native/third-party components that don't render in Jest (maps, camera, charts, scanners, printers) mocked at the module level in `jest.setup.ts` or `__mocks__/`, with a visible placeholder that carries the testID. |
| SHOULD | Forms tested through the real `react-hook-form` path (zod resolver or rules): fill fields with `userEvent.type`, submit, assert validation messages and the dispatched intent; a "press twice" test on steppers and dependent fields guards the React Compiler freeze (`rn-architecture-standards.md` §11). |

### 7. Integration tests

| Tier | Rule |
|---|---|
| MUST | At least one navigation-level test per feature: render the feature's navigator, drive from list → detail → back, asserting screen content and that params are ids (and merged on the way back). |
| MUST | Auth gating tested: unauthenticated store → auth stack rendered; authenticated → main stack. |
| SHOULD | Deep-link handling tested by rendering with an initial URL/state. |
| SHOULD | Persistence rehydration tested once: a persisted v(N-1) state passes through `migrate` and the app renders; navigation-state restore to a deep screen re-derives the non-persisted flags (`redux-saga-best-practices.md` §6). |

### 8. E2E — Maestro

```yaml
# .maestro/flows/todos-add.yaml
appId: ${APP_ID}
tags: [smoke]
---
- launchApp:
    clearState: true
- runFlow: ../common/login.yaml
- tapOn:
    id: "todos.add-button"
- inputText: "Buy milk"
- tapOn: "Save"
- assertVisible: "Buy milk"
```

| Tier | Rule |
|---|---|
| MUST | Flows live in `.maestro/flows/`, shared steps in `.maestro/common/`, the workspace config is `config.yaml` at the root of the flows workspace (`.maestro/config.yaml` when the workspace is `.maestro/`); `appId` and credentials from env, never literal. |
| MUST | Element selection by `id:` using the project's `testID` convention (§3) or by visible text; never by coordinates. |
| MUST | A `smoke` tag set runs on every release candidate build; the full set nightly or pre-release. Flows are run against the real EAS/Fastlane build artifact, not a dev client. Projects on the local-release profile (`rn-cicd-standards.md` §0 B) run the smoke set on the deliverable build before hand-over. |
| MUST | Test accounts and seed data are provisioned by a script or a test backend endpoint; flows don't depend on leftover state (`clearState: true`). Runtime-environment-selector apps (`rn-project-standards.md` §9 model B) need a flow step that enters the environment code first. |
| SHOULD | Keep flows linear and short; one assertion goal per flow; shared `login.yaml`, `onboarding-skip.yaml`. |
| SHOULD | Record a video/screenshots on failure in CI; attach to the run. |
| SHOULD | Existing Detox suite: keep if it is green and maintained; add new flows in Maestro; migrate opportunistically. Don't run two E2E frameworks forever — document the end state. The Level 0 device-driven workflow (accessibility CLI + `testID`) is the natural precursor: the same ids and flows move into Maestro files. |

### 9. Test quality rules

| Tier | Rule |
|---|---|
| MUST | Test names describe behavior: `it('shows retry when loading fails')`, not `it('works')` / `it('renders')`. |
| MUST | No `test.skip` / `it.only` committed; `eslint-plugin-jest` `no-focused-tests`, `no-disabled-tests` as errors (disabled requires a linked issue in a comment). |
| MUST | Flaky tests are quarantined within a day (tagged, excluded, issue opened) — retries in CI are not a fix. |
| MUST | Deterministic: seeded faker, fixed `Date` via `jest.setSystemTime`, no network, no real timers in saga tests. |
| SHOULD | `eslint-plugin-testing-library` enabled (`prefer-user-event`, `no-node-access`, `await-async-queries`). |
| SHOULD | Arrange/Act/Assert structure, one behavior per test, minimal mocking (mock at the boundary: API client, native modules, DB adapter, time). |
| SHOULD | A test for every bug fix (regression test named after the issue). At Level 0 the equivalent is a named manual check in the verification report and, when the bug class can recur silently, a lint rule or a repo-invariant script (`rn-docs-and-agents-standards.md` §3). |

---

## Part 2 — Audit prompt (Claude Code)

```text
<<<<< PROMPT START >>>>>

Audit this React Native repository's testing and verification setup against "the rules" in `rn-testing-standards.md` (Part 1). Read it first; stop and ask if missing.

Do not modify source or tests until I approve in Phase 5. Docs and the migration plan may be written in Phase 4.

## Phase 0 — Inventory
First determine the level: is there a test runner (jest config, `test` script, `*.test.*` files)? If none, say "Level 0" and inventory the Level 0 tooling; do not report the absence of Jest as thirty separate gaps.

Level 0 inventory (always):
- Static checks: scripts present, `--max-warnings 0`, repo-invariant scripts in pre-commit (list them); run them and report pass/fail with real output.
- `testID` coverage: sample interactive elements (`Pressable`, `TouchableOpacity`, `Button`, inputs, list items) and report the share with a `testID`; naming convention followed?
- Debug tooling: developer gate, debug-screen registry, debug screens per feature, mock-data folders, saga mock flags, flag override screen.
- Device-driven verification guide: present, dated, which tools (simulator CLI, accessibility CLI, adb), coordinate systems documented, tap-by-testID documented, known quirks list.
- Live-state and DB tooling: redux state CLI, DB extraction scripts, DDL regeneration, persisted-store location documented, logcat fallback documented.
- Verification reporting: does the agent-instructions file require a verification report; is there a manual checklist; is "no direct backend calls" stated.
- Is the level stated in the docs?

Level ≥ 1 inventory (when a runner exists), with paths and counts:
- Jest config: preset, setup files, moduleNameMapper vs tsconfig paths agreement, transformIgnorePatterns, coverage config and thresholds, scripts.
- Run `npm run test:ci` (or `test`) and report: pass/fail, duration, number of test files, coverage per metric, any `console.error`/act warnings in output, skipped/focused tests.
- Count tests per layer: unit (reducers/selectors/utils/migrations), saga (redux-saga-test-plan usage), component (RNTL `render`), integration (navigation container in tests), E2E (`.maestro/` flows, Detox `e2e/`).
- Test utilities: is there a `renderWithProviders`? factories? fake API client? testID catalog or convention? Count raw `render(` imports from RNTL in tests.
- RNTL version and usage: `getByTestId` vs role/text queries counts; `fireEvent` vs `userEvent`; whole-tree `toMatchSnapshot` count.
- Mocks: `jest.mock('fetch'|'axios')` or `global.fetch =` occurrences (should be zero); native module and DB adapter mocks present.
- Screens: list screens and whether each has tests for happy/loading/empty/error.
- Sagas: list watcher sagas and whether each has success/failure/cancel tests; DB-centric: registry and timestamp tests.
- E2E: framework, flows, tags, how they get credentials, whether they run in CI.
- Lint: eslint-plugin-jest / testing-library enabled?
- Flaky-test handling: retries configured in CI? quarantine list?
- Existing docs/agent files.

## Phase 1 — Confirm detect-rules
State and ask me to confirm: the current level and the target level for this audit (stay at 0 and harden it / move to 1 / higher), with your recommendation based on size, risk areas and team size; E2E tool (Maestro if none; keep Detox if present and green — ask whether to plan a migration); whether API mocking should move to a fake API client (vs existing module mocks); testID approach (typed catalog vs convention + grep); coverage threshold starting values (propose current numbers rounded down); which screens/sagas are highest priority for new tests (propose by user impact: sync, auth, persisted migrations first). Wait.

## Phase 2 — Audit
Each rule up to the target level: status, evidence/counts, tier, size S/M/L/XL, files, risk. Rules above the target level are listed once as "deferred by level", not audited.
Sizing: S = config/setup change or a docs statement; M = add utilities (renderWithProviders, factories, fake API) + migrate a handful of tests, or complete the Level 0 tooling (simulator guide, state CLI, registry); L = write tests for all screens or all sagas missing them, or testID coverage of the whole app; XL = E2E from scratch including CI device job, or replacing a large snapshot-based suite.
Report test-writing gaps as counts: "11 of 14 screens lack error-state tests", "6 of 9 watcher sagas untested", "testID on 38% of sampled interactive elements".

## Phase 3 — Gap report (chat)
1. Table MUST first; 2. totals; 3. top 5 by value/effort — Level 0 typically: state the level in docs, simulator guide with tap-by-testID, state CLI, testID coverage of the main flows, debug registry; Level ≥ 1 typically: jest.setup fixes that remove act warnings, renderWithProviders + fake API (unblocks everything), saga tests for auth/sync/registry, migration tests, one Maestro smoke flow in CI, fail-on-console; 4. accepted exceptions; 5. any currently failing or flaky tests, or a Level 0 project whose static checks are red, at the top.

## Phase 4 — Docs + plan (write now)
- Agent-instructions file: "Testing and verification" section — the level in one line, how to run the checks, the required utilities to use (paths) or the Level 0 tools (simulator guide, state CLI, DB extraction), query priority, userEvent rule, the four screen states, saga test template, testID convention, no snapshots, the verification report rule, no direct backend calls.
- Human docs: `testing.md` (or extend existing): the level and what it means, the pyramid for this project, utilities with examples from the codebase, the debug-first workflow, how to add a Maestro flow, how to handle flaky tests; at Level 0 also the simulator-interaction guide and the manual verification checklist if missing.
- `testing-migration-plan.md` in the docs location: gap table with checkboxes and counts; order: state the level → Level 0 tooling gaps → (if moving up) setup fixes → utilities → fail-on-console → saga tests (auth/sync/registry first) → migration tests → screen tests by priority → integration → E2E smoke in CI → coverage ratchet; accepted exceptions.
Show a diff summary.

## Phase 5 — Ask before applying
Offer "everything" / "all MUST" / "pick by §", with sizes/risk, and recommend. Default:
- Level 0: now — docs state the level, simulator guide, state CLI and DB scripts if missing, debug registry; then testID coverage module by module as part of feature work.
- Moving to Level 1: now — Jest setup, utilities (`renderWithProviders`, factories, fake API, `createTestStore`), lint plugins, fail-on-console (as `warn` first if there are many warnings); then, as separate PRs per feature: saga tests (sync and auth first), migration tests, reducer tests.
- Level 2/3: screen tests (happy + error first); E2E: one smoke flow + CI job as its own PR; migration of existing E2E only if the team agrees.
Wait; re-ask if ambiguous.

## Phase 6 — Apply what I approved
- Utilities first; then migrate existing tests to them with a codemod where mechanical.
- When writing new tests: follow query priority, userEvent, factories, fake API; name by behavior; one behavior per test; no snapshots.
- Run the suite after each item; a new test that is flaky on 3 consecutive runs is removed and reported, not retried.
- Never weaken an existing assertion to make a test pass; report the real failure instead.
- At Level 0: for every change, run the static checks, drive the affected screens on the simulator per the guide, read the live state, and write the verification report; never claim a device check you did not run.
- Tick the plan per item; finish with summary, coverage before/after (or testID coverage before/after at Level 0), and remaining gaps.

<<<<< PROMPT END >>>>>
```
