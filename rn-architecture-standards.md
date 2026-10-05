# React Native Architecture & Component Standards + Audit Prompt

> Standards version: 2 (2026-10-05). What changed: see `STANDARDS-INDEX.md` → Version history.

Part 1: rules (MUST / SHOULD). Part 2: audit prompt for Claude Code. Companion to `redux-saga-best-practices.md` and `rn-project-standards.md`.

Assumes: TypeScript, Expo or bare RN, **NativeWind** as the styling target (a legacy UI library mid-migration is handled, see §4), **react-hook-form** for forms (zod resolver preferred, see §7), Expo Router **or** React Navigation (the LLM detects which), **React Compiler** on or planned (§11).

Where a rule offers **two options**, a project picks one, records it in its docs and applies it consistently; the mix is the gap.

---

## Part 1 — Rules

### 1. Project layout

Two accepted layouts — **detect**:

**Option A — feature-first with barrels**

```
src/
  app/            # Expo Router routes only (thin: import a screen and render it)  [Expo Router]
  navigation/     # navigators, typed ParamLists, linking config                  [React Navigation]
  features/
    todos/
      components/ # feature-private components
      screens/    # screen components (TodoListScreen.tsx)
      hooks/      # feature hooks (useTodoList.ts)
      store/      # slice, sagas, selectors (see redux doc)
      api/        # feature API calls, typed
      types.ts
      index.ts    # public API of the feature only
  components/     # shared, generic UI (Button, Text, Screen, ListItem)
  hooks/          # shared hooks (useAppState, useDebounce)
  store/          # store setup, root saga, typed hooks
  config/         # env.ts, constants, feature flags client
  services/       # cross-cutting clients: api client, analytics, storage, i18n
  theme/          # tailwind tokens source, nativewind theme, typography scale
  utils/          # pure helpers, no RN imports
  types/          # ambient declarations
  assets/
```

**Option B — flat feature modules with a module map** (large apps with 50+ modules)

```
src/
  <featureName>/          # one folder per feature or infrastructure concern, directly under src/
    components/           # PascalCase.tsx
    screens/              # PascalCaseScreen.tsx (+ <Feature>DebugScreen.tsx)
    store/                # kebab-case-slice.ts, kebab-case-saga.ts, *-persisted-slice.ts
    interfaces/           # PascalCase.interface.ts
    database/             # kebab-case-db.ts (DB-centric profile)
    http/                 # kebab-case-api.ts, dto/, interceptors/
    hooks/ utils/ mock-data/
  core/                   # store, navigation, http client, themes — the app shell
  components/ui/          # vendored primitives (see §2)
  sharedComponents/       # app-level shared wrappers and screen building blocks
  util/ types/ …
```

Option B has no barrels; instead a **module map** document lists every folder with its purpose, screens, slices, tables, endpoints and entry points, and a pre-commit script fails when a folder has no entry (`rn-docs-and-agents-standards.md` §2).

| Tier | Rule |
|---|---|
| MUST | One of the two layouts, applied consistently. Option A: a feature imports from another feature **only** via its `index.ts`; enforce with `eslint-plugin-boundaries` or `import-x/no-internal-modules`. Option B: every `src/` folder has a module-map entry kept current by a script; cross-module imports are allowed but the map's "depends on" notes name them. |
| MUST | The shared UI folder holds only generic, feature-agnostic UI. If a component knows about `Todo`, it lives in the feature's `components/`. |
| MUST | `utils/` has no React or React Native imports (keeps them unit-testable with plain Jest, or at least reviewable in isolation). |
| MUST | Route files (`app/**` in Expo Router) contain no logic: `export default function Route() { return <TodoListScreen /> }` plus route options. Logic lives in the feature's `screens/`. |
| MUST | **One component per file**; file name = component name. Rare exception: a tiny private helper tightly coupled to the main component, which gets the same memoization treatment. |
| SHOULD | Max ~300 lines per component file; extract hooks/subcomponents beyond that. Debug screens, mock data and saga mocks live inside the feature (`screens/<Feature>DebugScreen.tsx`, `mock-data/`), not in a global debug folder (§12). |
| SHOULD | Legacy spellings and locations (older slice names, `mockData/` vs `mock-data/`) are listed in the naming doc with "do not rename, do not add new ones there"; a naming migration is never a side effect of a feature change. |

### 2. Components

Two accepted **component styles — detect**:

- **(A) function + named export**: `export function TodoCard({ todo }: Props) { … }`, `type Props = { … }` above; `React.FC` not used. Pairs with export style A (`rn-project-standards.md` §8).
- **(B) arrow + `React.FC` + `React.memo` at the default export**: `const TodoCard: React.FC<TodoCardProps> = ({ todo }) => { … }; export default React.memo(TodoCard);`. Inline props type for 2–3 props, a named `interface` (no `I` prefix) beyond that. Pairs with one-component-per-file and export style B. The legacy `TodoCardImpl` + `memo(TodoCardImpl)` variant is not used in new code and not refactored (DevTools reads the function name anyway).

With React Compiler on (§11), `React.memo` at the export is redundant for most components; a project may keep it as a convention for uniformity, and must not strip it from existing code in unrelated changes.

| Tier | Rule |
|---|---|
| MUST | One component style, applied consistently. |
| MUST | Three layers, in this dependency order: **screen** (owns data fetching/store wiring, no styling beyond layout) → **feature components** (receive data via props) → **primitives** (Button, Text, Input, Screen). Primitives never read the store. |
| MUST | A **Screen wrapper primitive** wraps every screen: safe area (`react-native-safe-area-context`; left/right only when the navigation header already consumes the top inset), background, keyboard avoiding (`KeyboardAvoidingView`, `behavior="padding"`, configurable vertical offset), a per-screen **error boundary** (§9), a `contentWrapper` choice (scroll view with `keyboardShouldPersistTaps="handled"`, or a fixed `flex: 1` box for self-scrolling content such as FlashList), `padding` (0 for full-bleed lists, gutter on the rows), and `topChildren` / `bottomChildren` slots for fixed bars. No raw `SafeAreaView` from `react-native`. Never add an `onStartShouldSetResponder` + `Keyboard.dismiss()` handler on the wrapper: under Fabric it runs for every touch and steals focus from inputs on iOS. |
| MUST | A `Text` primitive with typography variants is the only text component used; raw `react-native` `Text` is banned outside it (`no-restricted-imports`). Ensures font scaling, colors and a11y defaults are consistent. (Mid-migration projects: the legacy library's `Text` stays allowed in unmigrated files, see §4.) |
| MUST | Props are explicit; no spreading of unknown props into native components (`{...rest}` only onto a known base type like `PressableProps`). |
| MUST | No business logic in JSX: conditions and transforms are computed above `return` or in a hook. |
| MUST | **`testID` on every interactive or assertable element**, in new code and in any code touched: pressables, form fields, every modal/dialog action button, list containers and item cards. Not on static text, pure layout or decorative icons. Naming `scope.element` in kebab-case (`shop-list.search-input`); list items append the **stable business key, never the index** (`order.product-card.${productId}`). Shared components accept an optional `testID`, apply it to their root and derive child ids (`${testID}.ok`); the call site owns the scope; the vendored layer passes it through and never bakes ids in. `testID` complements `accessibilityLabel`, never replaces it; icon-only buttons need both. It is the hook for agent-driven device verification today (`rn-testing-standards.md` §0) and for E2E later. |
| MUST | **Vendored primitive layer** (copied-in shadcn / React Native Reusables components in `components/ui/`): keep upstream style (`function` declarations, explicit prop types, no manual `React.memo` — the compiler covers them, and `children`-taking primitives gain nothing from `memo`), strip web-only code on add (`Platform.select({ web })`, web class strings; lint-banned in native-only apps), spread `{...props}` so `testID` and a11y props flow through, explicit `import React` where the namespace is used in types. App-level wrappers around them follow the project's component style, not upstream's. |
| SHOULD | Compound components (`Card`, `Card.Header`) for shared primitives with slots; avoid boolean-prop explosions (`isPrimary isLarge isOutlined`) — use a `variant` union. |
| SHOULD | Every primitive has a Storybook (react-native-storybook) or an on-device playground / migration-gallery screen behind the developer gate (§12), so primitives are checked visually on both platforms after every upgrade. |
| SHOULD | `displayName` set when wrapping with `memo`/`forwardRef` in style A (stack traces and DevTools); style B gets it from the named const. |
| SHOULD | Long-press on a card with interactive children: a plain RN `Pressable` around the card, not a UI-library pressable whose children swallow the gesture. Status icons on rows inform, they do not gate: an informational icon may use a broader check than the edit/delete guard; do not unify them. |

### 3. Hooks & logic

| Tier | Rule |
|---|---|
| MUST | **Where screens read data — detect, two options.** (A) Through a feature hook (`useTodoList()`) returning `{ todos, status, actions }`, no scattered `useAppSelector` in the screen. (B) Thin components: `useAppSelector(selectX)` per value and `dispatch(actions.y())` directly, with **all business logic in slices and sagas** (`redux-saga-best-practices.md`); the component renders, selects, dispatches and holds local UI state only. Option B needs the "one selector per value, never an object-building selector" rule enforced in review. |
| MUST | Hooks follow the rules of hooks; `exhaustive-deps` is an **error** and deps are never silenced with a comment (with the compiler on, suppressing any `react-hooks/*` rule is lint-banned, §11). |
| MUST | No `useEffect` for derived state — compute inline or `useMemo` (or let the compiler do it, §11). `useEffect` is for synchronizing with external systems (subscriptions, native modules, store dispatch on focus). `react-hooks/set-state-in-effect` is an error. |
| MUST | Side effects tied to a screen use `useFocusEffect` (both libraries expose it), not `useEffect`, since screens stay mounted in the stack. |
| MUST | An unsaved-changes guard is **one shared hook** (`usePreventBackNavigation(navigation)` style) that intercepts only user-initiated back navigation and renders its confirm dialog; `beforeRemove` is lint-banned (`no-restricted-syntax`) because it intercepts programmatic `goBack()` too and is unreliable on native stacks. The hook's docs say to add an effect that navigates away once the modal is open and there are no changes left. |
| SHOULD | `useCallback`/`useMemo` only when a measured re-render problem exists, when passing to list props, or when the compiler cannot help (§11). Default is no memoization. |
| SHOULD | Async in hooks goes through the store/saga or a dedicated query layer; components never `await fetch` directly. |
| SHOULD | Hooks that subscribe to several contexts (theme, translation, store) are **not** called inside list render items; call them once in the list owner and pass values as props (§6). |
| SHOULD | A `useX` hook file documents its contract with JSDoc (inputs, what it dispatches, what it returns, the gotcha that made it exist). The project keeps a hooks catalogue in the docs. |

### 4. Styling — NativeWind (and the legacy library during a migration)

| Tier | Rule |
|---|---|
| MUST | One styling **target**: NativeWind `className`. `StyleSheet.create` and inline `style={{}}` only where NativeWind cannot express it (animated values, absolute measurements from layout) — and then with a comment. Enforce `react-native/no-inline-styles` as error. |
| MUST | **Mid-migration projects** (a legacy UI library such as NativeBase, Paper, Tamagui still in most files): rules are tagged per library (`[NB]`, `[RNR]`, `[both]`) in the docs; a "which library for new code" table decides per situation (new screen → target; new dialog even inside a legacy screen → target; small fix in a legacy file → stay legacy; a component family the target lacks on the branch → legacy). Migrating a whole screen is a planned step, never a side effect of a fix. The migration has one entry document with locked decisions, phases, risks and gotchas, and the agent-instructions file points to it. |
| MUST | Design tokens live in `tailwind.config.js` (`theme.extend`: colors, spacing, fontSize, borderRadius) as CSS variables. No raw hex/pixel values in `className` (`bg-[#ff0000]`, `w-[137px]`) — enforce via a lint rule or review; tokens are the source of truth. During a migration the tokens are **anchored to the legacy library's resolved colours** so migrated and unmigrated screens match; the token doc records the hex→HSL table and which legacy palette each token came from. |
| MUST | Dark mode via NativeWind `dark:` variants and the color scheme hook; no manual `colorScheme === 'dark' ? … : …` branching in components. NativeWind's `colorScheme.set` routes through native `Appearance`, so `userInterfaceStyle` in `app.json` must be `"automatic"`; a `"light"` lock silently no-ops dark mode. A persisted "dark mode" setting is synced into NativeWind (and the legacy library) in one place at the navigation root. |
| MUST | Variants via `cva` / `tailwind-variants` (`tv()`), not string concatenation of classes. |
| MUST | Prettier `prettier-plugin-tailwindcss` enabled so class order is deterministic. |
| SHOULD | Typography scale (`text-body`, `text-title`) defined as tokens and exposed only through the `Text` primitive's `variant` prop. Text does **not** inherit styles in RN (legacy libraries faked it); each `Text` is styled individually. |
| SHOULD | Spacing uses the 4-pt scale from tokens; no odd values. |
| SHOULD | `cssInterop` registrations for third-party components (vector-icon families, so icon colour is `className`-driven) live in one file (`theme/interop.ts` or `components/ui/icon.tsx`). One icon system; no second icon library because the component kit defaults to one. |
| SHOULD | **Gotchas documented where they bite**: a Tailwind class used for the first time may not render until Metro recompiles the CSS (restart Metro before judging a screenshot); colour-token opacity modifiers (`bg-primary/25`) need `<alpha-value>` in the token definition or they render unpredictably (use an explicit hairline `View` with a background instead); sticky list headers need `bounces={false}` / `overScrollMode="never"` on some list libraries; full-bleed lists put the gutter on rows. |

### 5. Navigation

Common rules, then per-library.

| Tier | Rule |
|---|---|
| MUST | Route params carry **identifiers only** (`{ todoId: string }`), never objects. The screen loads the entity from the store or DB by id. (Serialization, deep links, and state restoration all break with objects.) |
| MUST | Fully typed navigation: no `any` route names or params. |
| MUST | Deep linking configured and documented (`linking` config / Expo Router scheme); every screen that can be deep-linked validates its params with zod before use. |
| MUST | Auth gating in one place (root layout / root navigator), driven by store state; screens never check auth themselves. Stacks are grouped by availability (logged-out, logged-in-always-available, after-day-start, debug) and the navigation atlas names which stack every route is in. |
| MUST | **Going back to a screen with params merges**: `navigate` / `popTo` with params default to **replacing** the target's params and silently wipe what the screen was opened with; pass `{ merge: true }` unless resetting is the intent. Document it as a Critical Don't once it has bitten. |
| MUST | **Debug routes are gated at the entry point, not by registration.** A debug stack registered conditionally with `if (!__DEV__)` that does not `return` is a no-op; assume debug screens ship in release builds and gate their menu behind a developer role/flag (§12). Nothing on a debug screen may be unsafe to run against a production backend. |
| SHOULD | Navigation actions wrapped in a tiny typed helper per feature (`todosNav.openDetail(id)`) so string route names appear in one file. |
| SHOULD | Modals, sheets and alerts are routes or a single `OverlayHost`, not `useState` toggles scattered per screen. Shared dialog components (`ConfirmDialog`, `InfoDialog`) take a `testID` and translate their texts, with an escape hatch for raw titles. |
| SHOULD | **Navigation state persistence** (restore the stack after an app kill) is a documented decision; when on, entry actions do not re-fire on restore (`redux-saga-best-practices.md` §6). |
| SHOULD | A **navigation atlas** doc lists every registered route, its stack, who navigates to it and with what params; a navigation-flow change updates it. |

**Expo Router [detect]**
| Tier | Rule |
|---|---|
| MUST | `experiments.typedRoutes: true`; `href` and `router.push` use the generated types. |
| MUST | Layout files (`_layout.tsx`) only declare navigators and guards; no feature logic. |
| MUST | Group routes `(tabs)`, `(auth)` used to separate auth and main stacks. |
| SHOULD | `useLocalSearchParams` immediately parsed with a zod schema into typed params. |

**React Navigation [detect]**
| Tier | Rule |
|---|---|
| MUST | `RootStackParamList` and per-navigator `ParamList`s declared; `declare global { namespace ReactNavigation { interface RootParamList extends RootStackParamList {} } }` so `useNavigation` is typed without generics. |
| MUST | Navigators declared in `src/navigation/` (or `core/navigation/`), one file per navigator; `screenOptions` defaults centralized; native stack (`@react-navigation/native-stack`), the JS stack package lint-banned after the migration. |
| MUST | `NavigationContainer` has `linking`, `onReady` (hide splash, crash-reporter nav instrumentation), `onStateChange` (current screen name as a crash-reporter tag), and a `theme` derived from tokens. |
| MUST | After a major upgrade, the removed/renamed options of the previous major (`unmountOnBlur`, `headerBackTitleVisible`, `animationEnabled`, `tabBarOptions`, …) are **lint-banned via `no-restricted-syntax`** with the replacement in the message, so copy-paste from old tutorials fails at lint time. |
| SHOULD | Static navigation API (RN Navigation 7) preferred for new navigators: typing comes for free. |
| SHOULD | Day-start / login flows that swap the root navigator do it from an effect driven by state, not from a timer callback. |

### 6. Lists & performance

| Tier | Rule |
|---|---|
| MUST | Any list longer than a screen uses `FlashList` (or `FlatList` with `getItemLayout` when heights are fixed); `ScrollView` + `map` is banned for dynamic lists. FlashList v2 measures items itself: no `estimatedItemSize`. Inside the screen wrapper, use the fixed-box content wrapper and wrap the list in a `flex: 1` view. |
| MUST | List `data` is an array of **ids** (or stable references from a memoized selector / the current SQL page); the row component is its own memoized component file and selects its own entity by id. One row change must not re-render the list. Pass `extraData` when item contents can change without the array identity changing. |
| MUST | `keyExtractor` returns a stable domain id, never the index. |
| MUST | `renderItem` and other function props passed to lists are stable (`useCallback`, or compiler-memoized) — this is the one place memoization is mandatory. |
| MUST | **Lift hooks out of render items**: theme, translation and row-invariant store selectors are called once in the list owner and passed as props; each hook call in a render item runs per visible cell. |
| MUST | **Cell recycling breaks stateful controls**: a radio/checkbox from a UI library keeps internal state that is not reset when a recycled cell shows another item (taps land on the wrong item). Default fix: `key` from the item's business id so the control remounts per item. Lightweight `Pressable`-based controls are a documented exception for high-frequency lists only. |
| MUST | Images through `expo-image` (or `react-native-fast-image` on bare) with explicit `contentFit`, placeholder/blurhash, and sized sources; no unbounded remote images in lists. |
| MUST | **SQL-first for big data** (DB-centric profile): lists over ~1,000 rows are filtered, sorted and paged in the database, not in JS selectors; infinite scroll loads pages (`onEndReached` → load-more action, page size constant, footer spinner). `redux-saga-best-practices.md` §9. |
| SHOULD | Heavy screens profiled with React DevTools / Flashlight before optimizing; test lists with 1,000+ items on a low-end Android device; `why-did-you-render` available behind a dev flag. |
| SHOULD | Animations: Reanimated worklets for gesture/scroll-driven motion; `Animated` from core only for trivial fades. Layout animations opt-in per component. |
| SHOULD | Avoid anonymous objects/arrays in JSX props on hot paths inside list rows (`style={[a, b]}` recreated per render) unless the compiler is on and covers them. |
| SHOULD | Large dynamic forms (200+ fields with dependencies) subscribe each field to its transitive parents only (`useWatch` with a name list), never to the whole form; parent sets are computed once per form and cached. |

### 7. Forms — react-hook-form (+ zod)

Two accepted **validation styles — detect**:

- **(A) zod resolver**: schema in `features/x/schemas.ts` → `z.infer` type → `useForm({ resolver: zodResolver(schema) })`; the schema is shared with the API layer where the shapes match.
- **(B) RHF rules + validator factories**: `rules={{ required: …, validate: createIsWholeNumberValidator(isMandatory, t) }}` with the validators in one `*-validators.ts` per feature, messages through `t()`. Accepted in projects where zod is used for API payloads but forms predate it; new forms in such a project may still choose A, and the docs say which.

| Tier | Rule |
|---|---|
| MUST | One validation style per form, no hand-rolled `useState` validation. Messages are i18n keys; the keys exist in every locale file (`rn-runtime-quality-standards.md` §D). |
| MUST | Inputs are wrapped once in a `FormField`/`Controller` primitive that wires `value`, `onChangeText`, `onBlur`, error text and a11y props, and passes `testID`; screens never call `Controller` directly (projects whose forms predate the primitive keep `Controller` in old files and use the primitive in new ones). |
| MUST | **React Compiler rules for RHF** (§11): never read `watch('field')`, `formState` fields or `getValues()` during render; `useWatch({ control, name })` and `useFormState({ control })` instead. Children get `control` only — never `watch`, `formState` or the whole `formMethods` object. `getValues()` belongs in handlers and `validate` callbacks; handlers that need the current value take the `Controller` render-prop `value` as a call-time argument. Lint-enforced via `no-restricted-syntax` (destructuring `watch`, any `watch(` call, the `UseFormWatch` / `FormState` type imports, `watch=` / `formState=` JSX props). |
| MUST | Submit handlers are `handleSubmit(onValid)`; `onValid` dispatches an intent action or calls the feature hook — no API calls inside the component. Multi-step forms validate per step with `trigger(fieldNames)`. |
| MUST | Server-side validation errors are mapped back to fields with `setError` (field) or to a form-level error; never shown only as a toast. |
| MUST | **Clear a field with `''` (or `null`), never `undefined`**: `Controller` renders the **default value** whenever the stored value is `undefined`, so a field cleared with `undefined` keeps showing its default after `reset(data)` / `defaultValues` (edit screens, refill flows) while the saved value is gone. Validation and save mappers treat `''` as empty. |
| MUST | A standard `useForm` config is documented and copied (`mode`, `criteriaMode`, `shouldFocusError`, `reValidateMode`, always complete `defaultValues`); three complexity levels (single component; `FormProvider` + `useFormContext` sections with dot-notation names; dynamic forms keyed by question id with pre-registration) each have a reference file. |
| SHOULD | Keyboard handling (next-field focus, `returnKeyType`, keyboard-aware scrolling) handled by the Screen/form primitive, not per form. |
| SHOULD | **Draft persistence for long forms**: a hook saves `getValues()` to storage when the app goes to background/inactive and restores on the next foreground, returning a `clearSavedFormState` for the save path; keyed per form instance. |
| SHOULD | Conditional input adornments (a clear "x" only for non-empty text) keep the slot's **presence** stable while the field is focused; an adornment that appears mid-typing can remount the `TextInput` and drop focus in wrapper-based primitives. |

### 8. Data flow summary

| Tier | Rule |
|---|---|
| MUST | Unidirectional: screen → (feature hook →) store/saga → API or DB. A component never imports from `services/api` or a `*-api.ts` module directly. |
| MUST | Server state vs UI state separation (see redux doc §2). Form state lives in react-hook-form, not the store, unless it must survive navigation. |
| MUST | Loading/empty/error states are explicit in every screen that fetches; a shared `AsyncContent` / `CenteredScreenState` primitive renders them consistently. |
| SHOULD | Optimistic UI for user-initiated mutations with rollback (redux doc §2) in online profiles; local-is-truth in the DB-centric profile. |

### 9. Errors, boundaries, resilience

| Tier | Rule |
|---|---|
| MUST | `ErrorBoundary` at root (app-level fallback with "restart") and per screen (inside the Screen wrapper, which is one reason the wrapper is mandatory) reporting to the crash reporter with the component stack. |
| MUST | A typed `AppError` (`kind: 'network' \| 'auth' \| 'validation' \| 'notFound' \| 'unknown'`) is the only error shape crossing the API boundary; UI maps `kind` → message key (i18n). In a `catch`, `error` is `unknown`: one shared `getErrorMessage(error)` helper, never `error.message`. |
| MUST | Network-dependent screens handle offline explicitly (`@react-native-community/netinfo`), with a retry affordance. DB-centric apps work offline by construction and the docs' manual checklist includes an airplane-mode pass. |
| MUST | **Logging follows what production keeps.** When a Babel transform strips `console.*` in production (keeping `console.error`), code calls `console.log` directly without `if (__DEV__)`; an inner `if (!__DEV__) return` exists only where the function does real work besides the log (walking a payload, registering an interceptor). Per-area styled loggers (one colour each) are listed in the docs. Temporary debug logs use a searchable `[AREA DEBUG]` prefix and are removed with the fix. `console.error` is for real errors only, since it survives. (`rn-runtime-quality-standards.md` §B.) |
| SHOULD | Global handlers set at startup: `ErrorUtils.setGlobalHandler`, unhandled promise rejection tracking, both forwarding to the crash reporter. |
| SHOULD | Splash screen hidden only after store rehydration, DB initialization and initial auth check; a DB init failure is reported and still lets the app render rather than hang on the splash. |

### 10. Platform & native

| Tier | Rule |
|---|---|
| MUST | Platform branches via `Platform.select` or `.ios.tsx` / `.android.tsx` files, never scattered `Platform.OS ===` inside JSX. Native-only apps lint-ban `web` branches (`Platform.OS === 'web'`, `Platform.select({ web })`) so copied-in components lose their web code on add. |
| MUST | **Exactly one `GestureHandlerRootView`**, outermost in `App.tsx`. Under the New Architecture gesture recognizers mount only inside it; without it drawer swipes and backdrop taps silently stop working. **Never nest a second one**: on Android a nested root kills every tap, scroll and swipe app-wide with no error (iOS tolerates it). The one legitimate nested root is inside an RN `<Modal>` (a separate native window); portal-based overlays render into the main window and must not be wrapped. The upgrade checklist re-verifies this after every gesture-handler / screens / reanimated bump, **after a navigator remount**, not only on first launch. |
| MUST | Native modules / Expo modules are wrapped in a `services/` adapter with a typed interface and a mock for tests and Storybook. |
| MUST | Permissions requested lazily at the point of use with a rationale screen, through one `services/permissions.ts` helper. |
| MUST | **Native-level fixes are patches, not forks.** `patch-package` patches with a doc entry per patch (why, hunks, when it can go), regenerated only through the project's regen script (`rn-project-standards.md` §2); a patch that grows with every RN bump is listed as debt with its exit plan (usually a library migration). |
| MUST | **Both platforms are verified** for touch handling, borders, modals and selection colour; the manual checklist says so. Known per-platform traps live in the component docs (single-side borders that do not render on Android inside some pressables, a select inside a form control opening the keyboard on Android, always-mounted modals going unresponsive on iOS until early-returned when closed). |
| SHOULD | New Architecture enabled (Fabric/TurboModules) for RN ≥ 0.76 / current Expo SDK; libraries that don't support it are tracked as debt. The generated native folders are checked for the expected flags after a prebuild. |
| SHOULD | Hermes enabled (default), no reliance on non-Hermes-safe APIs (e.g. `Intl` edge cases documented). Known dev-only toolchain crashes (debugger attach bugs) are documented with the RN version that fixes them, so nobody debugs app code for them. |
| SHOULD | **DB-centric apps:** after a SQLite schema change, cold-restart the dev app (terminate + launch) — Fast Refresh runs new queries before DB init re-runs. |

### 11. React Compiler

Applies when `babel-plugin-react-compiler` is on (Expo: `experiments.reactCompiler: true` in `app.json`) or planned. The compiler auto-memoizes components, hooks, derived values and callbacks, which changes what is safe to call during render and what the lint rules mean.

| Tier | Rule |
|---|---|
| MUST | `eslint-plugin-react-hooks` ≥ 7 with the compiler rules as **errors**: `set-state-in-effect`, `preserve-manual-memoization`, `refs`, `immutability`, `incompatible-library` (plus the `recommended` set: `purity`, `globals`, `static-components`, `use-memo`, `error-boundaries`, `unsupported-syntax`, …). `rule-suppression` as `warn` (it is not in `recommended`; it only fires for suppressions the compiler itself detects). |
| MUST | **Never suppress a `react-hooks/*` rule.** A suppression makes the compiler silently skip that component (no error, no memoization); the bug shows up as a stale value weeks later. Ban it with `@eslint-community/eslint-comments/no-restricted-disable: ["error", "react-hooks/*"]` plus `no-unlimited-disable` (a bare `/* eslint-disable */` would bypass the first). Legacy suppressions live in one file-scoped override at the bottom of the lint config, each with a comment naming the file and the reason; the list may only shrink. |
| MUST | **Subscription-on-read APIs freeze.** Any library API whose return value depends on hidden mutable state read during render (react-hook-form `watch()`, `formState` proxy reads, `getValues()`; similar patterns in other form, store or animation libraries) gets memoized by the compiler at its first value: "works once, then never again" — handlers compute from the frozen value while the visible input stays fresh because the library's own components are not compiled. Use the library's hook-based subscription (`useWatch`, `useFormState`), pass only stable handles (`control`) to children, and lint-ban the hazardous forms with `no-restricted-syntax` (`react-hooks/incompatible-library` catches only the same-component case). The agent-instructions file lists these as a numbered Critical Rule. |
| MUST | **Do not add `useMemo` / `useCallback` by reflex** in new code; inline handlers and object literals in JSX are fine. **Keep existing** `useMemo` / `useCallback`: do not strip them in unrelated changes (`preserve-manual-memoization` errors when the compiler cannot preserve one). `React.memo` at the export stays if the project's component style uses it (§2); vendored primitives do not need it. |
| MUST | A **health check** script (`react-compiler-healthcheck` over `src/`) is in the scripts and run after dependency upgrades and before enabling the compiler on a new folder; its output (compiled vs. bailed-out components, incompatible libraries) is recorded in the migration plan. |
| SHOULD | When the compiler lands in an existing project: enable it, run the health check, fix every bailout and `incompatible-library` hit before trusting it, then audit for render-time subscription reads (grep `watch(`, `getValues(`, `formState.` across components). The RHF freeze has bitten real apps in steppers and dependent fields; test "press twice" flows on device. |
| SHOULD | Verify third-party hooks and styling interop (NativeWind `className` → style, animation hooks) under the compiler on a component that re-renders on a theme/dark-mode change, once per major upgrade. |

### 12. Developer tooling inside the app

Debug screens, mock data and on-device switches are part of the architecture, not an afterthought; they are how a feature is verified when the backend is late and how an agent drives the app (`rn-testing-standards.md` §0).

| Tier | Rule |
|---|---|
| MUST | **One developer gate**: a settings entry visible only to users with a developer role/flag opens a developer-settings screen that lists every debug screen from **one registry** (`DEBUG_SCREENS: { route, label, category }`), grouped and searchable. Adding a debug screen = route type + stack registration + registry row; the checklist is in the docs. |
| MUST | Debug screens ship in release builds (§5) and may be reached on production backends; nothing on them is unsafe to run there without its own guard. Inserting mock rows into a synced table is dev-only and always paired with a clear action on the same screen. |
| MUST | **Every feature flag and server-config field is toggleable on a debug screen** (`rn-runtime-quality-standards.md` §E), so a feature can be tested without a backend change; overrides are temporary (reset on the next fetch). |
| MUST | **Saga-level mocks for endpoints that are not ready** (`redux-saga-best-practices.md` §4): dev-settings flags (one per endpoint, optional outcome enum), typed mock data in the feature's `mock-data/` with edge cases, switches on the feature's debug screen. |
| MUST | Debug screens follow the same rules as app screens (screen wrapper, `testID`s, one component per file), except that their labels may be plain untranslated strings. |
| SHOULD | A feature's debug screen shows row counts of its tables, lets you insert/clear mock data, and exposes the feature's persisted state; a sync debug screen shows the in-flight upload set and timestamps; a "debug info" screen shows version, build, environment, flag values. |
| SHOULD | The agent-instructions file carries the rule "when implementing or changing a feature, ask about debug needs if the task does not specify them: mock data, a debug screen, saga mocking, a flag override" (`rn-docs-and-agents-standards.md` §5). |

---

## Part 2 — Audit prompt (Claude Code)

```text
<<<<< PROMPT START >>>>>

Audit this React Native repository's architecture and component conventions against "the rules" in `rn-architecture-standards.md` (Part 1). Read it first; stop and ask if missing.

Do not modify source code until I approve in Phase 5. Documentation and the migration plan file may be written in Phase 4.

## Phase 0 — Inventory
Report with file paths and counts:
- Navigation library (Expo Router or React Navigation, version); typed routes / ParamLists; linking config; where auth gating happens; stacks and how debug routes are gated; navigation state persistence on/off; `navigate`/`popTo` with params without `{ merge: true }`; `beforeRemove` usages; lint bans for removed navigation options.
- Styling: NativeWind present and version; a legacy UI library present (which, how many files import it); tailwind.config tokens and whether they carry `<alpha-value>`; count of `StyleSheet.create` usages, inline `style={{` usages, raw hex/px values in classNames; dark-mode approach and `userInterfaceStyle`; a migration entry doc if mid-migration.
- Folder layout: option A or B; top-level src folders; cross-feature deep imports (option A) or module-map coverage (option B, run its check script if present).
- Components: style A or B (count `React.FC` vs `function` components, default vs named exports), raw `Text`/`SafeAreaView` imports from react-native, components > 300 lines, a vendored primitive folder and whether it keeps upstream style, `testID` count vs interactive-element count (sample `Pressable`/`TouchableOpacity`/`Button`/inputs without `testID`).
- Hooks: data-reading option A or B; count of `eslint-disable.*react-hooks`, `useEffect` usages that set state from props/other state, `useEffect` vs `useFocusEffect` in screens; a shared unsaved-changes hook.
- React Compiler: enabled? (`experiments.reactCompiler`, babel plugin); `eslint-plugin-react-hooks` version; which compiler rules are errors; is suppressing `react-hooks/*` banned; grandfathered files; health-check script present and last result; grep for render-time `watch(` / `getValues(` / `formState.` in components and `watch`/`formState` passed as props.
- Lists: `ScrollView` + `.map(` patterns, `FlatList`/`FlashList` usage and version, `estimatedItemSize` on FlashList v2, `keyExtractor` returning index, unmemoized `renderItem`, hooks called inside render items, stateful UI-library controls inside list rows without a per-item `key`.
- Forms: validation style A or B counts; a `FormField` primitive; `Controller` directly in screens; fields cleared with `undefined`; draft-persistence hook.
- Navigation params: any `navigate(..., { someObject })` passing objects rather than ids; `any` in ParamLists.
- Errors: error boundaries present and where; a typed AppError or string errors; `getErrorMessage`-style helper; console stripping transform and `if (__DEV__)` wrappers around logs.
- Images: `Image` from react-native vs expo-image/fast-image.
- Platform: `Platform.OS ===` occurrences in JSX (and `web` branches in a native-only app); `GestureHandlerRootView` count and nesting; native module wrappers; New Architecture flag; Hermes; `patches/` contents and regen script.
- Developer tooling: developer gate, debug-screen registry, debug screens per feature, mock-data folders, saga mock flags, flag override screen.
- Existing docs / agent-instruction files and their style; a navigation atlas; a hooks catalogue.

## Phase 1 — Confirm detect-rules
State what you found and will apply for: navigation library; layout option (A/B); component style (A/B); data-reading option (A/B); styling target (confirm NativeWind is the target even if a legacy library dominates today — if NativeWind is absent, ask whether to adopt it or audit against "StyleSheet + theme" instead) and, mid-migration, the "which library for new code" table; forms validation style (A/B) for new forms; export/default-export conventions; React Compiler status (on / to enable / off by decision); which primitives already exist (Screen wrapper, Text, Button, FormField, AsyncContent) vs need creating. Ask me to confirm in one message and wait.

## Phase 2 — Audit
For every rule (keep §numbering): status ✅/⚠️/❌/➖, evidence with counts, tier, size S/M/L/XL, files touched, risk low/med/high.
Sizing: S = under 1h; M = half a day (e.g. introduce a Text primitive and codemod simple cases, add the react-hooks suppression ban with a grandfather list); L = a few days (feature-folder restructure, list rewrites, NativeWind migration of a few screens, testID coverage of a module); XL = week+ (full StyleSheet→NativeWind migration, enabling the compiler on a large app with many bailouts, navigation library migration — the latter is never recommended by this audit, only noted).
Use counts from Phase 0 as the size driver. Never claim compliance without evidence.

## Phase 3 — Gap report (chat)
1. Table: rule → tier → status → size → files → risk; MUST first.
2. Totals for "all MUST" and "all MUST + SHOULD".
3. Top 5 by value/effort. Typically: Screen + Text primitives (unlocks several rules at once), react-hooks suppression ban + RHF render-time read ban when the compiler is on, ids-only route params + `merge: true` audit, list virtualization + memoized rows + lifted hooks, debug-screen registry and flag overrides, typed AppError.
4. Accepted-exception candidates (e.g. a legacy screen that will be rewritten anyway, a documented `Pressable`-based control in a high-frequency list).
5. Performance and correctness red flags found (ScrollView+map over large data, unbounded images, nested GestureHandlerRootView, frozen RHF reads under the compiler) go at the top.

## Phase 4 — Docs + migration plan (write now)
- Agent-instructions file (CLAUDE.md/AGENTS.md; create if none): "Architecture conventions" section — layout option, component style, the primitive components to use (and their paths), styling rule (NativeWind only, tokens only, which library for new code mid-migration), route-params-ids rule and `merge: true`, forms rule incl. the compiler-safe RHF APIs, list rule, error rule, one `GestureHandlerRootView`, testID rule, debug-tooling question. Short bullets; recurring traps as numbered rules.
- Human docs: extend the existing docs location or create `architecture.md` (full rules adapted to real paths, one small example per section taken from the codebase), `components.md` (catalog of primitives with props, per-platform traps), a hooks catalogue, a navigation atlas, and a debug-screens guide. Link from README.
- `architecture-migration-plan.md` in the docs location: date, detect-decisions, gap table with checkboxes and counts, suggested order (primitives first → lint bans (suppressions, RHF reads, nav options, web branches) → debug registry → lists → forms → styling migration per screen → folder moves last, since they churn imports), accepted exceptions.
Show a diff summary.

## Phase 5 — Ask before applying
Offer "everything" / "all MUST" / "pick by §" with sizes and risk, and recommend. Default recommendation:
- Now: create missing primitives (Screen, Text, FormField, AsyncContent) + lint rules banning raw imports (set to `warn` first); the react-hooks suppression ban with a grandfather list; `no-restricted-syntax` bans for RHF render-time reads, removed nav options and web branches; typed ParamLists; AppError type; debug-screen registry. These are additive and low risk.
- Separate PRs per feature: list rewrites, forms migration, NativeWind migration screen-by-screen (never a big-bang restyle), testID coverage per module.
- Folder restructure as one mechanical PR after boundaries lint or the module-map check is in, using the editor/`tsc`-checked moves, when the team has a quiet window.
- Enabling the React Compiler on an existing app: its own PR after the health check is clean.
Wait; re-ask if ambiguous.

## Phase 6 — Apply what I approved
- One §item or one feature per commit; conventional commit messages in the project's convention; tick the plan.
- Additive first: new primitives land unused, then codemods adopt them.
- Codemods for mechanical replacements (`jscodeshift` or `ast-grep`), never hand-editing dozens of files.
- After each item: `npm run typecheck && npm run lint` (+ `npm test` when a runner exists); for UI changes, describe what to visually verify on device and run it on the simulator when the project has an agent-driven device workflow (`rn-testing-standards.md` §0); restart Metro before judging a screenshot that uses a new Tailwind class.
- Never change navigation structure and styling in the same commit.
- Stop and ask when an item is larger or riskier than estimated.
Finish with a summary and updated plan totals.

<<<<< PROMPT END >>>>>
```
