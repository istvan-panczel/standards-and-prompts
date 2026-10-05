# Project Standards — Index & Run-All Prompt

> Standards version: 2 (2026-10-05). Version history at the bottom.

Drop this folder (or just the files) into a React Native repo under `docs/standards/` (or the project's docs tree, or the gitignored scratch folder while evaluating) and use the prompts in Claude Code.

| File | Covers | Audit output files (in the project's docs location) |
|---|---|---|
| `rn-project-standards.md` | Node/npm pinning, dependency policy (Expo SDK, patches), tsconfig (max strict, `noUncheckedIndexedAccess` recipe), ESLint 9 flat (compiler rules, suppression ban, `no-restricted-syntax` hazards), Prettier, hooks + repo-invariant scripts, commit conventions (scoped / work-item), releases, aliases + export style, env models (build-time / runtime selector), build config + signed deliverables, repo hygiene | `project-standards.md`, `release.md`, scripts guide, `project-standards-migration-plan.md` |
| `rn-docs-and-agents-standards.md` | The documentation system: entry file (CLAUDE.md), rule files, module map + check, doc link check, follow-up files, plans, agent working rules (plan first, ask before forks, git, no backend calls, verification report), task-file slash commands, scratch folder | entry file, `module-map.md`, `open-follow-ups.md`, `upgrade-followups.md`, check scripts, `docs-migration-plan.md` |
| `redux-saga-best-practices.md` | Store, slices, selectors (reselect 5 param selectors), sagas (`SagaReturnType`, effects, mocks), persistence (whitelist, migrations, reset/refresh), RN specifics (navigation-persistence trap), tooling (live-state CLI), API & data layer, **three project profiles** incl. DB-centric offline-first (read-only synced DB, push-then-pull registry, sync timing, wipe paths) | `redux-saga-conventions.md`, sync-orchestration and database docs (profile C), `redux-saga-migration-plan.md` |
| `rn-architecture-standards.md` | Layout (feature-first / flat modules + map), components (two styles, testID, vendored primitives), hooks, NativeWind + mid-migration rules, navigation (merge params, debug gating, nav-option bans), lists & perf, forms (RHF compiler-safe APIs, clear with `''`), errors & logging, platform (one gesture root, patches), **React Compiler**, **in-app developer tooling** | `architecture.md`, `components.md`, hooks catalogue, navigation atlas, debug-screens guide, `architecture-migration-plan.md` |
| `rn-testing-standards.md` | **Verification levels 0–3** (Level 0: static checks + debug tooling + device-driven verification + testID coverage), Jest setup, test utilities, unit/saga/component/integration, Maestro E2E, test quality | `testing.md`, simulator-interaction guide (Level 0), `testing-migration-plan.md` |
| `rn-cicd-standards.md` | **Delivery profiles** (CI-driven / local release), pipeline shape, CI reproducibility + secrets, EAS/Fastlane builds + signing verification + key inventory, **multi-tenant builds**, OTA, release pipeline, E2E in CI, quality gates, workflow hygiene; GitHub Actions and Azure Pipelines references | `release.md` (extends), `cicd-migration-plan.md` |
| `rn-runtime-quality-standards.md` | Security (secure storage, federated login, server-side cert checks, deep links, logging models), Sentry (dev model, auto-reported list, saga pattern) & analytics, accessibility, i18n (two layouts, key-integrity scripts, typed dynamic keys), feature flags (vendor / backend-driven two-tier, debug overrides) | `SECURITY.md`, `observability.md`, `accessibility.md`, `i18n.md`, `FLAGS.md` or field reference, `runtime-quality-migration-plan.md` |

Every file has the same shape: **Part 1** rules tiered MUST / SHOULD (with `[RN]` / `[C]` / `[detect]` tags), **Part 2** an audit prompt that inventories → confirms detect-rules with you → audits with S/M/L/XL sizes and file counts → reports gaps → writes docs + a migration plan with checkboxes → asks what to apply → applies with verification.

## How the standards handle real projects

- **Detect-rules with two options.** Where two conventions are both defensible (component style, folder layout, commit header, formatter wiring, export style, env model, i18n layout, flag source, logging model, Sentry dev model, delivery profile, verification level), the rule names both, the project picks one and records it, and the audit treats **mixing** as the gap, not the choice. Rules stay MUST where a project is simply behind (engine-strict, lockfile + `npm ci`, `--max-warnings 0`, parameterized SQL, one gesture root).
- **Declared levels and profiles.** A project declares its testing level (`rn-testing-standards.md` §0), its delivery profile (`rn-cicd-standards.md` §0) and its data profile (`redux-saga-best-practices.md` §9). Rules above the declared level are deferred, not gaps; an **undeclared** absence is the gap.
- **Documentation goes where the project keeps it.** The docs standard detects the docs tree (`docs/`, `claude/`, …) and every other audit's Phase 4 writes there. `docs/` in the tables above means "the detected location".
- **Anonymous.** The standards name no company, customer or product; patterns learned in one project are stated as rules with their reason.
- **Agents are an audience.** Every file's Phase 4 writes a short section into the agent-instructions file; `rn-docs-and-agents-standards.md` defines that file's shape so the seven sections land in one place.

## Recommended order (first time)

1. `rn-project-standards.md` — everything else assumes the scripts, pinning, lint and hooks it sets up.
2. `rn-docs-and-agents-standards.md` — decides where docs live and what the entry file looks like; **run before any other file's Phase 4**. Its module map is also how the other audits find the code.
3. `redux-saga-best-practices.md` — data layer, profile and store conventions that architecture and tests build on.
4. `rn-architecture-standards.md` — primitives, compiler rules and layout that tests and runtime quality reference.
5. `rn-testing-standards.md` — declares the level the CI standard depends on.
6. `rn-cicd-standards.md` — needs the scripts (1), the level (5) and the release tool (1).
7. `rn-runtime-quality-standards.md` — security items are urgent, but the services it introduces fit cleanly only after 3–4; run its **Phase 0 security scan early anyway** (see prompt below).

Each audit is a few hours of LLM time plus your review; applying the recommended "now" items from all seven is typically a week of focused work on a mid-sized app, and the L/XL items are a backlog.

## Run-all prompt

Use when you want one session to orchestrate everything (long session; a per-file run gives you more control).

```text
<<<<< PROMPT START >>>>>

This repository has a set of standards files: `rn-project-standards.md`, `rn-docs-and-agents-standards.md`, `redux-saga-best-practices.md`, `rn-architecture-standards.md`, `rn-testing-standards.md`, `rn-cicd-standards.md`, `rn-runtime-quality-standards.md` (find them; if any is missing, list which and ask whether to continue without it). Each contains rules (Part 1) and an audit prompt (Part 2) with phases.

Run them as follows:

1. First, run ONLY the security portion of `rn-runtime-quality-standards.md` Phase 0 (secrets in repo/bundle, token storage, cleartext transport, deep-link validation, debug-signed deliverable paths, secrets written into committed files by a pipeline) and report any critical/high findings before anything else. If a committed secret is found, stop and tell me to rotate it.

2. Detect the project's declarations before auditing, and confirm them with me in ONE message: documentation location and entry file (docs standard §1), testing level 0–3 (`rn-testing-standards.md` §0), delivery profile A/B (`rn-cicd-standards.md` §0), data profile A/B/C (`redux-saga-best-practices.md` §9), check-script names, commit convention, React Compiler status. Wait.

3. Then run the seven audits in this order: project → docs-and-agents → redux-saga → architecture → testing → cicd → runtime-quality. For each: execute its Phase 0 (inventory), Phase 1 (confirm detect-rules — batch all seven files' Phase 1 questions into ONE message to me, grouped by file, and wait), Phase 2 (audit) and Phase 3 (gap report). Rules above the declared level or outside the declared profile are listed once as deferred, not audited. Produce ONE consolidated gap report in chat: a table per file, then a cross-file totals section (MUST gaps, SHOULD gaps, total size, files touched), and a cross-file top 10 by value/effort that accounts for dependencies (e.g. primitives before a11y fixes, scripts before CI, module map before the other docs, suppression ban before compiler trust).

4. Execute every file's Phase 4 (docs + migration plan): write all documentation into the project's documentation location found in step 2, one agent-instructions file (the entry file; create CLAUDE.md if none) with one short section per area, and one migration plan per file. Also create `standards-migration-overview.md` in the docs location linking the seven plans with their totals and a suggested cross-file order. Show a diff summary of everything written.

5. Ask me once what to apply across all files: "everything" / "all MUST" / "pick by file+§" / "the cross-file recommended set", with your recommendation (default: all security critical/high, then every file's "now" set as described in its Phase 5, in the dependency order above). Wait for my answer.

6. Apply what I approved following each file's Phase 6 rules, in dependency order, one §item per commit (only if I asked you to commit; otherwise leave the changes for me), running the project's check scripts (`typecheck`, `lint`, `format:check`, the repo-invariant scripts, and `test` only if a test runner exists) after each, ticking the relevant migration plan as you go. At testing Level 0, verify UI changes on the simulator per the project's guide and report what you verified. Stop and ask when anything is larger or riskier than estimated. Finish with a consolidated summary, updated totals in the overview, open security items, pending decisions, and manual steps I still need to do.

Throughout: never modify source code before step 6; never print, commit or move secrets; never weaken tests or static checks to make them pass; never call the project's backend directly; keep documentation in the repo's existing style and location; reference code by path and symbol, not line number; no personal or machine data in docs.

<<<<< PROMPT END >>>>>
```

## Re-audit prompt (later)

```text
Re-audit this repository against the standards files (same seven as `STANDARDS-INDEX.md`). Read each migration plan in the docs location, re-check every unchecked item and spot-check checked ones, re-confirm the declared level and profiles, update the plans' statuses and totals and `standards-migration-overview.md`, then report what regressed, what was completed outside the plan, and the current top 5 remaining items. Do not change source code; ask if you think something should be applied now.
```

## Maintaining the standards

- A rule proven wrong for a project becomes an **accepted exception** in that project's docs, not a silent edit to the standards file. A rule proven wrong for **several** projects, or too narrow (one convention where two are defensible), is changed here and becomes a detect-rule with both options.
- When a rule changes in these files, bump the `Standards version:` line at the top of **every** file (they are versioned together), add an entry to the version history below, and note it in the project's `standards-migration-overview.md` on the next re-audit.
- Library versions referenced (ESLint 9, typescript-eslint 8, eslint-plugin-react-hooks 7, RNTL 12+, React Navigation 7, Expo SDK current, NativeWind 4) will drift; the prompts verify versions in Phase 0 rather than trusting these files. Known drift at the time of version 2: `tseslint.config()` is deprecated in favour of `defineConfig`; `eslint-plugin-react-native-a11y` declares ESLint ≤ 8 peers; RNTL 14 moved to the `test-renderer` peer; `redux-saga-test-plan` is unmaintained since 2022.

## Version history

### 2 (2026-10-05)

Reworked after running the set against a large offline-first Expo app (React Compiler on, SQLite source of truth, no test runner, local releases, multi-tenant builds, Azure DevOps). What changed:

- **New file** `rn-docs-and-agents-standards.md`: the documentation system (entry file, rule files, module map + check script, doc link check, follow-up files, plans), agent working rules and the task-file slash-command workflow. All other audits write their docs where it says.
- **Detect-rules with two accepted options** replaced single-convention MUSTs for: component style (function/named vs `React.FC` + `React.memo` default), folder layout (feature-first vs flat modules + module map), commit header (scoped vs work-item `#id`), formatter wiring (`eslint-config-prettier` vs `eslint-plugin-prettier`), export style, environment model (build-time env vs runtime environment selector), screen data access (feature hook vs thin components), forms validation (zod resolver vs RHF rules), i18n layout (namespaces vs one file per language + scripts), feature-flag source (vendor vs backend-driven two-tier), production logging model (logger service vs Babel strip), Sentry dev model (on vs off in dev), root saga vs per-saga run.
- **React Compiler** is a first-class section (`rn-architecture-standards.md` §11) and shows up in the lint baseline: compiler rules as errors, a hard ban on suppressing `react-hooks/*`, the subscription-on-read freeze (react-hook-form `watch` / `formState` / `getValues`) with `no-restricted-syntax` bans, memoization guidance, the health-check script.
- **Testing** gained verification levels; Level 0 (no runner, by decision) is a declared state with its own MUSTs: static checks, repo-invariant scripts, testID coverage, debug tooling, a device-driven verification guide, live-state and DB extraction tooling, the verification report, no direct backend calls.
- **CI/CD** gained delivery profiles (CI-driven / local release), a hard rule against debug-signed deliverables with a signature-verification script, a signing-key inventory, the multi-tenant build section (config module, apply script, one keystore per app id, Firebase project vs store), the stale-pipeline rule, and Azure Pipelines equivalents.
- **Redux/saga** gained the DB-centric offline-first profile (read-only synced DB, backend-driven by design, push-then-pull registry contract, sync timing invariant, full vs delta sync, wipe paths, UI lock, batch validation, database and schema discipline), the persistence rules learned the hard way (whitelist key = reducer key, migration chain semantics, default changes need a migration, reset/refresh on every persisted slice, transient state reset in refresh), the navigation-persistence trap, reselect 5 parameterized selectors, `SagaReturnType` as MUST, the saga-level mock pattern, the live-state CLI and its `--dispatch` limit, the validation-mode decision (strict vs report-only).
- **Project standards** gained the Expo dependency policy (`expo-doctor`, `expo install --fix`, never `@latest` for SDK-managed), the patch-package regen rule (exclude build artifacts), the `noUncheckedIndexedAccess` fix recipe, the `exactOptionalPropertyTypes` decision, repo-invariant scripts in pre-commit, `no-restricted-syntax` for project hazards, docs in `format:check`, the scripts guide, the scratch folder, CNG discipline, the tool-attribution decision for commits.
- **Architecture** gained the testID convention, the vendored-primitive layer, mid-migration styling rules and gotchas (Metro restart for new classes, alpha tokens, dark mode `userInterfaceStyle`), `merge: true` on back-navigation params, debug-route gating, nav-option lint bans, the shared back-guard hook, list rules (lift hooks, cell recycling, FlashList v2), form rules (clear with `''`, draft persistence, compiler-safe RHF), one `GestureHandlerRootView`, cold restart after a schema change, and §12 in-app developer tooling (developer gate, debug-screen registry, flag overrides).
- **Runtime quality** gained the federated-login and server-side certificate-check rules, the "what is reported automatically" list, the saga catch pattern, the i18n key-integrity scripts and typed dynamic-key maps, the backend-driven flag tier with debug overrides and the field-reference table, and the gate taxonomy (flag / tenant check / role).
- **Prompts** no longer assume `npm test`, `docs/` or GitHub Actions; they detect the level, the profile, the docs location and the CI platform first, and batch the detect questions.

### 1

Initial six files.
