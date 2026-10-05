# React Native Runtime Quality Standards + Audit Prompt

> Standards version: 2 (2026-10-05). What changed: see `STANDARDS-INDEX.md` → Version history.

Security · Observability · Accessibility · i18n · Feature flags

Part 1: rules (MUST / SHOULD). Part 2: audit prompt for Claude Code. Companion to the other `*-standards.md` files.

Assumes: **Sentry** for crashes/performance; analytics vendor **detected** ("none" is a valid, recorded answer); **i18next + react-i18next**; feature-flag source **detected** (a vendor such as LaunchDarkly / PostHog / Statsig / GrowthBook / remote config, **or** the app's own backend configuration, §E option B).

---

## Part 1 — Rules

### A. Security (OWASP MAS-aligned)

| Tier | Rule |
|---|---|
| MUST | **Secure storage**: tokens, refresh tokens, PII at rest only in Keychain/Keystore (`expo-secure-store` or `react-native-keychain`); never AsyncStorage/MMKV unencrypted, never redux-persist. A `services/secureStorage.ts` adapter is the only access point (an auth saga being the single reader/writer is an accepted form). |
| MUST | **No secrets in the bundle**: anything in JS is public. API keys for third parties are either public-by-design (Sentry DSN, analytics write key) or proxied by the backend. Audit via `grep` for `sk_`, `secret`, `private`, Bearer literals, and by inspecting `expo export` / the release bundle. **Runtime-environment-selector apps** (`rn-project-standards.md` §9 model B) keep backend URLs out of the bundle entirely: they arrive in an encrypted payload the app validates before storing; the decryption key in the app is a tamper deterrent, not a secret, and the docs say so. |
| MUST | **Transport**: HTTPS only; `NSAllowsArbitraryLoads` false and no cleartext in Android network security config except explicitly allowlisted dev hosts in dev builds only. A backend-provided URL that is plain `http` on a bare IP is reported as an open item against the backend, not worked around in the app (iOS ATS will block it; an Android side effect that lets it through is not a fix). Internal links opened from backend content go through the system browser by default (`Linking.openURL`), not an in-app WebView, unless the content is trusted. |
| MUST | **Auth lifecycle**: short-lived access token in memory (store) + refresh token in secure storage; refresh single-flight (redux doc §6); logout wipes secure storage, store, persisted state, and caches (images included if sensitive). Federated login (Firebase + Microsoft/Google): the login result comes from the sign-in call's resolved credential, never from an auth-state listener that can replay the **previous** persisted user; every session-ending path signs out of the identity provider and a sign-out failure never blocks the logout; the backend logout is called only while the token is still in state and only online; a 401 on the logout request itself never triggers the session-expired flow. |
| MUST | **Signing-certificate checks are server-side.** Firebase Auth (and Play Integrity / App Attest) validate the app's signing certificate at sign-in against fingerprints registered in the console. `auth/invalid-cert-hash` is fixed by registering the **release** key's fingerprints, not by a rebuild; the **debug keystore's fingerprint is never registered** (it is public; anyone could sign a passing build). Local debug builds therefore use e-mail/password login, and the docs say so (`rn-cicd-standards.md` §3). |
| MUST | **Deep links** validated: every incoming URL/param parsed with zod; auth-required routes gated after validation; no navigation to arbitrary URLs from link params (open-redirect). Universal Links / App Links (verified domains) over custom schemes for auth callbacks. Outgoing deep links to partner apps (`scheme://open`) are documented with the partner's result codes. |
| MUST | **Logging/PII**: no PII, tokens, or request bodies in `console`, Sentry breadcrumbs, or analytics. One of two logging models (§B): a `logger` service with redaction and `console.*` lint-banned outside it, **or** a Babel transform that strips `console.*` from production (`exclude: ['error']`) with the rule "never log credentials even in dev, `console.error` is for real errors only". Sentry `beforeSend`/`beforeBreadcrumb` scrub headers and known PII fields; the redux enhancer's `stateTransformer` redacts token and password fields, and a new slice holding secrets is added to it (checklist item). |
| MUST | **WebView**: `javaScriptEnabled` only when required, `originWhitelist` restricted, no `injectedJavaScript` with untrusted data, `onShouldStartLoadWithRequest` allowlist. |
| MUST | **Dependencies**: `audit-ci` in CI with an allowlist (cicd doc §7); no abandoned native libs for security-sensitive features (crypto, auth, storage). |
| SHOULD | **Certificate pinning**: decision documented per app. Default: not pinned (operational cost, key rotation risk) unless the threat model requires it (fintech/health) — then pinned via `react-native-ssl-pinning` or native config with a rotation plan and a kill switch. |
| SHOULD | **Device integrity**: decision documented. Default: detect jailbreak/root and emulator (`jail-monkey` or Play Integrity / App Attest) and **log** rather than block, unless regulated. |
| SHOULD | **Screen protection** for sensitive screens: `expo-screen-capture` prevent / `FLAG_SECURE`, blur on app switcher (iOS) for banking-type data. |
| SHOULD | **Biometric gate** for returning to sensitive sections via `expo-local-authentication`, with a fallback and a timeout policy. |
| SHOULD | Obfuscation (ProGuard/R8 on Android, Hermes bytecode) enabled for release; understood as hardening, not security. |
| SHOULD | A `SECURITY.md` with the threat model summary, the decisions above, the signing-key inventory pointer, and the vulnerability reporting path. |

### B. Observability — Sentry + analytics [detect vendor]

| Tier | Rule |
|---|---|
| MUST | Sentry initialized first thing in app entry (`Sentry.init` before `registerRootComponent`/`App`, root exported as `Sentry.wrap(App)`), with `environment`, `release` (= `<app>@<native app version>`) and `dist` (= native build number, from `expo-application`) set so crashes map to a build. |
| MUST | **Dev builds — decide and record.** Two accepted models: (A) Sentry on in dev with a `development` environment and `tracesSampleRate: 1.0`; (B) **Sentry off in dev** (`if (!__DEV__) Sentry.init(...)`) so every `Sentry.*` call is a no-op locally and the console / logcat is the only error output. Model B keeps the dashboard clean and avoids dev noise, at the price that nothing Sentry-related can be tested without temporarily removing the guard (the code comment says so, and that change is never committed). Whichever model, the testing doc states where dev errors show up. |
| MUST | Source maps uploaded per release in CI (`@sentry/react-native` Metro plugin + `sentry-cli` in the release pipeline, cicd doc §5); unsymbolicated releases are a finding. Local verification builds set `SENTRY_DISABLE_AUTO_UPLOAD=true`. The DSN is configuration, never copied into docs. |
| MUST | Error boundaries report to Sentry with component stack (architecture doc §9); `AppError`s carry `kind` as a tag; expected errors (validation, 404 on user-typed ids) are **not** sent. |
| MUST | Navigation instrumentation enabled (Expo Router or React Navigation integration, `Sentry.reactNavigationIntegration()` passed to the container) so transactions and breadcrumbs carry screen names; `onStateChange` also sets the current screen name as a tag and its params as context. |
| MUST | **What is reported automatically is listed in the docs**, so feature code does not duplicate it: failed HTTP requests (interceptor with url, headers, data as extras), redux state via the enhancer (redacted), render errors (per-screen boundary), schema validation failures (a `safeParseAndReport` helper), navigation tags, user/tenant/environment/backend-version tags set at login and cleared at logout/reset. |
| MUST | **Saga catch pattern**: `console.error(error)` + `Sentry.captureException(error)` (+ `{ tags: { feature, stage }, extra }` only when it helps triage) + a failed action. `Sentry.captureMessage('[#<ticket>] what happened', { extra })` for non-exception anomalies worth knowing (unexpected backend payload shapes), with the ticket in the message so it is searchable. `Sentry.addBreadcrumb` for step tracing in multi-step flows. A small local tag vocabulary (`feature`, `stage`) is enough; a project-wide taxonomy is not required, but whatever exists is written down. |
| MUST | PII scrubbing configured (§A); `sendDefaultPii: false`; user context limited to an opaque id. |
| MUST | Analytics events, when an analytics vendor exists, go through one typed `services/analytics.ts` with an **event catalog** (`events.ts`: name + zod/TS schema per event); components call `track(Events.todoCreated, { source })`, never a vendor SDK directly. Vendor swap = one file. "No product analytics" is a valid state when recorded. |
| MUST | Consent respected: analytics (and Sentry session replay if used) initialized only after consent where required (GDPR/ATT); a `consent` state drives SDK enable/disable. |
| MUST | **Production logging model — detect** (one of): (A) a structured `logger` service (levels, dev console transport, prod Sentry-breadcrumb transport at `info`+, redaction) with `console.*` lint-banned outside it; (B) `babel-plugin-transform-remove-console` in the production env with `exclude: ['error']`, so `console.log/info/warn/debug` and their argument expressions vanish from release bundles. Under B: call `console.*` directly — **no `if (__DEV__)` wrappers** (an inner `if (!__DEV__) return` exists only where the function does real work besides logging, such as walking a payload or registering an interceptor); per-area styled loggers (`%c` prefix, one colour each) are listed in a table; temporary debug logs carry a searchable `[AREA DEBUG]` prefix and are removed with the fix; `console.error` survives and is for real errors only. |
| SHOULD | Performance monitoring on: app start, navigation transactions, slow/frozen frames (`enableNativeFramesTracking`); `tracesSampleRate` low in prod (0.1–0.2), 1.0 in preview/dev. |
| SHOULD | Alerting: a Sentry alert rule for crash-free sessions dropping below a threshold per release, routed to chat; release health enabled. |
| SHOULD | A "debug info" screen in non-prod builds (or behind the developer gate): version, build, fingerprint, env, feature-flag values, last Sentry event id, log level toggle; a "Sentry testing" screen that throws / captures on demand (model B makes it useful only with the dev guard lifted). |
| SHOULD | Event naming convention documented (`object_action` snake_case, past tense for completed actions), with a review rule: no new event without a catalog entry. |
| SHOULD | **Dev-only redux logging** (`redux-logger`, collapsed, diff) through a logger that truncates arrays, added in `__DEV__` only; the on-device DevTools plugin and the live-state CLI are documented together (`redux-saga-best-practices.md` §7). |

### C. Accessibility

| Tier | Rule |
|---|---|
| MUST | Every interactive element has an `accessibilityRole` and a meaningful `accessibilityLabel` (or visible text); icon-only buttons always have a label **and** a `testID` (the two are complements, `rn-architecture-standards.md` §2). Use the React Native props (`accessibilityLabel`), not `aria-*`, with UI libraries that do not forward ARIA aliases (NativeBase logs warnings and ignores `aria-label`). Enforce with `eslint-plugin-react-native-a11y` (`has-valid-accessibility-descriptors`, `has-accessibility-hint` where relevant); its peer range stops at ESLint 8, so load it in flat config through `@eslint/compat` `fixupPluginRules` and verify the rules fire — if they do not, review enforces and the docs say so. |
| MUST | Touch targets ≥ 44×44 pt (`hitSlop` or padding); primitives (`Button`, `IconButton`) enforce it so screens don't have to. |
| MUST | **Dynamic type**: the `Text` primitive respects `allowFontScaling` (default true) with sane `maxFontSizeMultiplier` (e.g. 1.5–2) only where layout would break; layouts tested at 200% (iOS Larger Accessibility Sizes). |
| MUST | Color contrast ≥ 4.5:1 for text (3:1 large text) in both light and dark themes; tokens in `tailwind.config` are checked once, not per screen. Mid-migration, the legacy library's resolved palette is checked the same way (and is what the tokens are anchored to). |
| MUST | Screen-reader flow: focus moves to the new screen's title on navigation (`accessibilityAutoFocus` / `AccessibilityInfo.setAccessibilityFocus`), modals trap focus (`accessibilityViewIsModal`), loading states announce (`AccessibilityInfo.announceForAccessibility` or live regions). |
| MUST | State conveyed without color alone: errors have icons/text; selected/disabled use `accessibilityState`. |
| MUST | Tests query by role/label first (testing doc §6) — a11y regressions fail tests. At testing Level 0, the device-driven verification reads the accessibility tree, so missing labels show up as unnamed elements in the agent's dumps and are fixed on the spot. |
| SHOULD | Reduced motion respected (`useReducedMotion` from Reanimated / `AccessibilityInfo.isReduceMotionEnabled`) for non-essential animations. |
| SHOULD | A manual a11y checklist per release (VoiceOver + TalkBack pass on the smoke flows) in `release.md`; the iOS Accessibility Inspector audit run on new screens. |
| SHOULD | `accessibilityLanguage` set when content language differs from device language. |
| SHOULD | Text selection and highlight colours are checked on Android for readability (a UI library's default selection colour can make selected text unreadable; the fix may have to be a patch when the library's theme resolution wins over app overrides). |

### D. Internationalization — i18next + react-i18next

Two accepted **resource layouts — detect**:

- **(A) Namespaces per feature**: `todos.json`, `auth.json`, `common.json` per language, lazy-loadable, keys `object.action`.
- **(B) One namespace, one file per language** (`assets/locales/en.json`, `hu.json`), bundled as `resources`, keys nested per feature in `SCREAMING_SNAKE_CASE` (`SHOP_LIST.TITLE`) with flat shared words at the root (`CANCEL`, `SAVE`, `REQUIRED_FIELD`). Fits apps with two or three languages and a few thousand keys; must be paired with the parity and unused-key scripts below.

| Tier | Rule |
|---|---|
| MUST | i18n initialized at startup with `expo-localization` (or `react-native-localize`) detecting device locale, **or** — documented decision — the language taken from a persisted user setting (business apps whose users work in one language regardless of device locale) synced into i18next in one place; a fallback language either way. |
| MUST | **No hardcoded user-facing strings** in components: `t('todos.addButton')` via `useTranslation`; enforced by `eslint-plugin-i18next` (`no-literal-string`, flat config supported) with allowlists for technical strings, or — where the plugin is not adopted — the review checklist item "user-facing text through `t()`". Debug screens may use plain untranslated labels. |
| MUST | **Key integrity is checked by tooling**, one of: typed keys (`declare module 'i18next' { interface CustomTypeOptions { resources: typeof en } }` so `t('typo')` is a type error), or a **parity script** in pre-commit that fails when a key exists in one language file and not the others, plus an on-demand **unused-keys script** (reports keys no source references, lists template-built keys separately as "possibly dynamic", `--strict` for CI). Layout B without these scripts is a gap. |
| MUST | **Dynamic keys are typed maps, not template strings.** `t(\`FEATURE.STATUS.${status}\`)` has no compile-time link to the files and renders the raw key when a leaf is missing. Use `const KEYS = { open: 'FEATURE.STATUS.OPEN', … } satisfies Record<Status, string>` and `t(KEYS[status])`: exhaustive by type, and every key stays statically discoverable by the unused-keys script. |
| MUST | Plurals and interpolation via i18next (`count`, `{{name}}`), never string concatenation; `Intl`-based formatting for dates, numbers, currencies (`Intl.DateTimeFormat`/`NumberFormat` — Hermes supports them; verify with a test), or `date-fns` locales. Date helpers never mix UTC and local parts in one formatted string. |
| MUST | Error messages are keys: `AppError.kind` → i18n key mapping, not server strings shown raw (server messages only as a last-resort fallback for `unknown`). Toast texts are keys resolved in the toast saga; a pre-translated string may pass through where `t()` returns non-keys unchanged, and the docs say that this is relied upon. |
| MUST | **Key naming conventions** are written down: `ToTranslate` suffix on props that carry a key instead of text; navigation titles translated at render time; tenant-specific texts use a separate key chosen in code, never a per-tenant file fork. |
| SHOULD | RTL readiness: `I18nManager` handled, logical layout props (`start`/`end`, NativeWind `ps-`/`pe-`), icons with direction mirrored; at least one RTL smoke test if an RTL locale is on the roadmap. Deferred (and said so) for apps whose language set is fixed and LTR. |
| SHOULD | A pseudo-locale (`en-XA` style, lengthened + accented) available in dev builds to catch truncation and hardcoded strings. |
| SHOULD | Translation files are the single source: extraction via `i18next-parser` in `validate` (fails on missing/unused keys); translators work via a platform (Crowdin/Lokalise) that round-trips the JSON. Layout B projects with in-house translation use the parity/unused scripts instead. |
| SHOULD | Locale-aware assets (images with text, legal docs) handled through the same key system, not `if (lang === …)`. |
| SHOULD | A glossary of the business terms per language in the docs (the words tickets use → the module they mean), because feature names in tickets are rarely the English identifiers in code. |

### E. Feature flags [detect source]

Two accepted **flag sources — detect**:

- **(A) Vendor**: LaunchDarkly, PostHog, Statsig, GrowthBook, Firebase Remote Config.
- **(B) Backend-driven, two tiers**: a **server configuration** endpoint (`mobile/config`-style) that says what is enabled for this tenant/company (`isStoreModuleEnabled`, limits, names), fetched after login and at day start, **persisted** so the last config works offline; and **user features** from the login response that say what this user may do (roles such as `SUPERVISOR`, `DEVELOPER`). "Is the feature on for the company?" vs "does this user have permission?"; some features need both.

| Tier | Rule |
|---|---|
| MUST | One typed access point: (A) `services/flags.ts` with a `Flags` type, a `useFlag('newTodoList')` hook and **defaults** for every flag; (B) one slice + selectors per tier (`selectIsXModuleEnabled`, `selectIsUserSupervisor`), **every boolean compared with `=== true`** so a missing field means off, fields optional with defaults, a failed fetch keeps the previous config and only changes the request state. The app is correct with the source unreachable. |
| MUST | Flags are evaluated once per session (or on source-pushed updates) and stored in Redux/state; components never call the vendor SDK or read the raw DTO. Selector names follow the **app's** concept, and when a backend field name differs from the app name (a legacy field name, a mapper rename), the mapping table lives in the docs so nobody guesses the selector from the field. |
| MUST | **Every flag is overridable on a debug screen** (`rn-architecture-standards.md` §12): a config debug screen with a form field per server-config field and a user-features toggle screen; overrides are temporary (reset on the next fetch/login). Adding a flag = DTO + schema + mapper (if renamed) + selector + **debug form field**, as a numbered checklist. |
| MUST | Every flag has an owner and an expiry/cleanup date recorded in `flags.ts` comments or a `FLAGS.md` (option A); option B records per field what it controls and where it is read (a field-reference table), since backend flags rarely expire but do get re-purposed. |
| MUST | Kill switches for risky surfaces (payments, sync, new native integrations) exist and are tested once (flag off → feature hidden, no crash). |
| MUST | **Flags gate, tenant checks gate, roles gate — the docs say which one a module uses.** Some older features check the tenant in code where a server flag would do; that is accepted and listed. A module without a gate is not "one tenant's feature": keep it working for every tenant when you change it. |
| SHOULD | Flag evaluations attached to Sentry (tags) and analytics context so crashes/metrics can be split by variant; option B sets tenant, environment and backend version as tags at login. |
| SHOULD | Flags are not configuration: environment-specific values (API URLs) live in env or the environment selector (`rn-project-standards.md` §9), not in flags; business parameters the backend owns (limits, names, dates) may ride on the server config. |

---

## Part 2 — Audit prompt (Claude Code)

```text
<<<<< PROMPT START >>>>>

Audit this React Native repository against "the rules" in `rn-runtime-quality-standards.md` (Part 1): security, observability, accessibility, i18n, feature flags. Read it first; stop and ask if missing.

Do not modify source until I approve in Phase 5. Docs and the migration plan may be written in Phase 4.

## Phase 0 — Inventory
Report with paths and counts, per area:

Security
- Token/PII storage: grep for AsyncStorage/MMKV/redux-persist usage with keys like token/auth/user; secure-store/keychain usage; is there a single adapter or single saga?
- Secrets in bundle: grep src and config for key-like strings; list third-party keys present and classify public-by-design vs should-be-proxied; environment model (build-time env vs runtime selector) and whether backend URLs are in the bundle.
- Transport config: `NSAppTransportSecurity`, `network_security_config.xml`, `usesCleartextTraffic`; any backend-provided plain-http endpoints and how links from backend content are opened (WebView vs system browser).
- Auth: where tokens live, refresh logic or session-expired flow, logout cleanup scope, federated login handling (listener vs sign-in result), 401 handling on the logout request.
- Signing checks: Firebase/attestation in use? Is the debug keystore fingerprint registered anywhere (ask if not visible)? Does the docs' key inventory exist (`rn-cicd-standards.md` §3)?
- Deep links: scheme/universal links config; param validation; any `Linking.openURL(param)`; outgoing partner deep links documented.
- Logging: production logging model (logger service / babel strip / neither); `console.*` count outside a logger; `if (__DEV__) console` wrappers count under the babel model; Sentry `beforeSend` present; redux enhancer `stateTransformer` redaction; what PII could leak.
- WebView usage and its props.
- Pinning / integrity / screen protection / biometrics: present or not, and whether a decision is documented.

Observability
- Sentry: init location and order, dev model (on / off behind `__DEV__`), `release`/`dist`/`environment`, source-map upload in CI or local disable flag, navigation integration, `onStateChange` screen tags, `sendDefaultPii`, sample rates, error boundaries reporting, HTTP failure interceptor, schema-failure helper, release health/alerts (ask if not visible in repo); is "what is reported automatically" documented?
- Analytics vendor(s) detected or "none"; is there a typed `analytics` service and event catalog? Count direct vendor SDK calls in components.
- Consent handling.
- Debug info / Sentry testing screen present?

Accessibility
- `eslint-plugin-react-native-a11y` enabled and actually firing under flat config? Count interactive elements lacking `accessibilityRole`/label (grep Pressable/TouchableOpacity without accessibility props; icon-only buttons); `aria-*` props used with a library that ignores them.
- `allowFontScaling={false}` and `maxFontSizeMultiplier` usage; Text primitive behavior.
- Contrast: compute ratios for token pairs in `tailwind.config` (text on background, both themes) and, mid-migration, for the legacy palette; list failures.
- Focus management on navigation/modals; `accessibilityState` usage; color-only state indicators; Android selection colour.
- Query style in tests (role/label vs testID counts) or, at Level 0, whether the device workflow dumps the accessibility tree.

i18n
- i18next present and version; init, locale source (device vs persisted setting), fallback; resource layout A or B; typed resources declared?
- `eslint-plugin-i18next` enabled? Count literal user-facing strings in JSX (sample-based estimate is fine; report method).
- Key integrity tooling: parity script, unused-keys script, where they run; count of template-literal keys (`t(\`` with `${`).
- Namespaces/keys structure and naming conventions documented; concatenation patterns (`+ t(`), manual plurals, raw `toLocaleDateString` vs Intl/date-fns; UTC/local mixing in date helpers.
- Error message handling (server strings shown raw?); toast text path.
- RTL/pseudo-locale/extraction tooling present; a business-term glossary.

Feature flags
- Source A (vendor) or B (backend-driven: server config + user features); typed access point and defaults; `=== true` comparisons for booleans; direct SDK/DTO reads in components; flag inventory with owners/expiry or a field-reference table with field→app-name mapping; debug override screens and whether every flag is on them; kill switches; which modules are gated by tenant checks vs flags vs roles (is it documented?).

Also: existing docs (`SECURITY.md`, a11y/i18n docs, logging doc, Sentry doc) and agent-instruction files.

## Phase 1 — Confirm detect-rules and decisions
State and ask me to confirm: analytics vendor or "none"; flag source (A/B); production logging model (A/B); Sentry dev model (A/B); i18n resource layout (A/B) and locale source; the three documented-decision items (certificate pinning, device integrity, screen protection) — propose a default per the rules and the app's domain; whether i18n is multi-language today or single-language (if single, rules D still apply but RTL/extraction are deferred — confirm); consent requirements (EU users? ATT?). Wait.

## Phase 2 — Audit
Each rule (keep §letters/numbers): status ✅/⚠️/❌/➖, evidence/counts, tier, size S/M/L/XL, files, risk.
Sizing: S = config/init change or a docs decision; M = introduce a service (secureStorage, logger, analytics, flags) + migrate a handful of call sites, or add the parity/unused-key scripts and the debug override fields; L = migrate all call sites (console→logger, literal strings→t(), vendor calls→service), a11y pass across all screens; XL = full i18n retrofit of a large app without i18n, or auth storage redesign with migration of existing users' sessions.
Security findings are reported with severity (critical/high/medium/low), not just size.

## Phase 3 — Gap report (chat)
1. **Security findings first**, by severity, with remediation and whether a secret must be rotated or a fingerprint de-registered.
2. Table MUST first; 3. totals; 4. top 5 by value/effort — typically: secure storage adapter + token move, Sentry release/dist + source maps + "what is reported automatically" doc, logging model decision + redaction, key-integrity scripts or typed keys, every flag on a debug screen, a11y lint + Text/Button primitive fixes; 5. accepted exceptions / documented decisions.

## Phase 4 — Docs + plan (write now)
- Agent-instructions file: "Runtime quality" section — use the storage/logger/analytics/flags access points (paths), the logging model in one line (no `__DEV__` wrappers under the babel model; no `console.*` outside the logger under the service model), Sentry dev model in one line, no literal strings, keys in every locale file, dynamic keys as typed maps, a11y props required on interactive elements (with `testID`), event catalog rule, no PII to Sentry/analytics, deep-link params validated, every new flag gets a debug form field, "redact new secret-holding slices in the state transformer".
- Human docs: `SECURITY.md` (threat model summary, decisions, signing-key pointer, reporting); `observability.md` (Sentry setup and dev model, what is reported automatically, saga catch pattern, loggers table, debug screens); `accessibility.md` and `i18n.md` (or one `runtime-quality.md` if the project keeps docs small; i18n includes the key conventions, scripts and the dynamic-key rule); `FLAGS.md` or the server-config field-reference table (field, app name, what it controls, where it is read) plus the "how to add a flag" checklist. Extend existing docs where they exist.
- `runtime-quality-migration-plan.md` in the docs location: gap table with checkboxes/counts; order: security critical/high → services and decisions (secureStorage, logging model, analytics, flags) → Sentry release mapping + source maps + auto-reporting doc → key-integrity tooling → flag debug coverage → a11y lint + primitives → i18n lint + typed keys / typed maps → call-site migrations per feature → documented-decision items → nice-to-haves; accepted exceptions.
Show a diff summary.

## Phase 5 — Ask before applying
Offer "everything" / "all MUST" / "pick by §" with sizes/risk/severity and recommend. Default:
- Immediately: critical/high security items (token storage, exposed secrets + rotation, cleartext transport, deep-link validation, debug fingerprint de-registration).
- Now: new services with defaults (additive), Sentry config and the auto-reporting doc, parity/unused-key scripts, debug override fields, lint rules at `warn`.
- Per feature PRs: console→logger (service model) or `__DEV__`-wrapper removal (babel model), literals→t(), template keys→typed maps, vendor calls→services, a11y fixes; lint rules flipped to `error` when counts hit zero.
- Decisions requiring product/legal input (pinning, integrity, consent) as documented defaults pending confirmation — not implemented silently.
Wait; re-ask if ambiguous.

## Phase 6 — Apply what I approved
- Security first, one finding per commit, with a note on any secret that needs rotation (never rotate or print secrets yourself) or fingerprint that must be removed from a console (tell me; you cannot do it).
- Services land additive, then codemods migrate call sites (`ast-grep`/`jscodeshift`), then lint flips to error.
- i18n extraction: generate keys from existing literals with a deterministic naming scheme that follows the project's layout (A or B), keep the source language resource as source, add the key to **every** language file (the parity script enforces it), never machine-translate silently.
- A11y: fix primitives first so most screens inherit; list remaining per-screen issues.
- After each item: `npm run typecheck && npm run lint` (+ `npm test` when a runner exists); describe what to verify on device with VoiceOver/TalkBack and at 200% text size, and run the device workflow where the project has one.
- Tick the plan; finish with a summary, open security items, decisions still pending, and manual steps (Sentry alert rules, vendor dashboards, consent copy, console fingerprint changes).

<<<<< PROMPT END >>>>>
```
