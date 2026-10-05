# React Native Project Standards & Audit Prompt

> Standards version: 2 (2026-10-05). What changed: see `STANDARDS-INDEX.md` → Version history.

Two parts, same shape as `redux-saga-best-practices.md`:

1. **Standards** — tooling and repository rules for React Native (Expo or bare), TypeScript, npm. Tiered **MUST** / **SHOULD**. Config snippets are reference baselines; the project's existing choices win where the rules say "detect".
2. **Audit prompt** — paste into Claude Code. It inventories the repo, reports gaps with effort sizes, writes/extends docs, leaves a migration plan file, and asks before changing anything.

Where a rule says **detect**, the LLM keeps whatever the project already uses if it is one of the accepted options, and only adds what is missing. Several rules below list **two accepted options**; a project picks one, records it in its docs, and applies it consistently. Mixing the two inside one repo is the gap, not the choice itself.

---

## Part 1 — Standards

### 1. Node & npm version pinning

| Tier | Rule |
|---|---|
| MUST | Node version is pinned in a file the whole team's tooling reads. **Detect** and keep one of: `.nvmrc` / `.node-version`, Volta (`"volta": { "node", "npm" }` in package.json), mise/asdf `.tool-versions`. If none exists, add `.nvmrc` (and mirror to `.node-version` — some tools read only one). |
| MUST | `"engines": { "node": ">=X.Y <Z", "npm": ">=A" }` in package.json **and** `engine-strict=true` in `.npmrc`, so `npm install` fails on the wrong version instead of silently proceeding. A lone `.nvmrc` protects only developers who run `nvm use`; the engines check protects everyone, CI included. |
| MUST | `package-lock.json` is committed; CI and fresh clones use `npm ci`, never `npm install`. |
| MUST | A strict `npm install` resolves with **zero peer conflicts**. No `legacy-peer-deps` in `.npmrc` — it hides incompatible native module pairs until runtime. |
| SHOULD | `save-exact=true` in `.npmrc` for packages the project manages itself. For Expo-managed packages keep the ranges `npx expo install` writes (`~57.0.22`): the lockfile pins them and `expo-doctor` validates them (see §2 dependency policy). |
| SHOULD | Use an LTS Node major that the current RN/Expo SDK supports; record the reason for the pin in the README ("Node 22 LTS, required by RN 0.8x / Expo SDK 5x"). |
| SHOULD | `packageManager` field (`"packageManager": "npm@10.x.y"`) so Corepack-aware tooling and CI pick the right npm. |

### 2. package.json hygiene & dependency policy

| Tier | Rule |
|---|---|
| MUST | A stable set of check scripts exists and is the only entry point CI, hooks and agents call. **Detect** the names: canonical `typecheck`, `lint`, `lint:fix`, `format`, `format:check`, `test`, `test:ci`; an existing project may keep its spelling (`lint-fix`, `format-check`) as long as every doc, hook and agent file uses the same names. Teams must not have to remember raw tool invocations. |
| MUST | `private: true` for apps. |
| MUST | **Expo dependency policy:** every Expo-managed or peer-managed package stays on the version `npx expo-doctor` accepts, patch bumps included. Align with `npx expo install --fix`; never `npm install <pkg>@latest` for an SDK-managed package, even when `npm outdated` shows a newer "Latest". Packages Expo does not manage (`eslint`, `prettier`, independent native libs) may move ahead; run `expo-doctor` afterwards. Record the policy and the current SDK/RN/React versions in one dated doc section. |
| MUST | **Patched dependencies** (`patch-package`): `postinstall: patch-package`; every patch is named with the exact installed version and has a doc entry (why, which hunks, when it can go). Patches are regenerated through a project script that excludes build artifacts (`.cxx`, `.gradle`, `build/`, `.DerivedData`, `.generated`, `apple/Products/`), never raw `npx patch-package <pkg>` — a raw run bakes megabytes of compiled output into the patch. After regenerating, `grep "^diff" patches/<file>.patch` must list source files only. Any bump of a patched package re-checks that the patch applies. |
| SHOULD | `scripts.validate` (or `check`) runs typecheck + lint + format check (+ tests when a runner exists, see `rn-testing-standards.md` §0) in one go; this is what the pre-push hook and CI call. |
| SHOULD | Dependencies are in the right bucket: build/lint/test tooling in `devDependencies`; nothing in `dependencies` that only runs at dev time. |
| SHOULD | **Scripts guide**: when the project grows past roughly 15 scripts, a `NPM_SCRIPTS.md` (or a section of the docs) lists every script with *what it does*, *when to use it* and *prerequisites*, grouped by purpose. Dev tooling scripts (DB extraction, live state, device helpers, build wrappers) live in a `utils/` (or `scripts/`) folder as Node `.mjs` files with a header comment, and get their own ESLint block with node globals so IDE linting does not flag `process`/`console`. |
| SHOULD | A documented upgrade checklist for SDK bumps: root providers and gesture root re-verified after a navigator remount, generated native folders checked for the expected architecture flags, local Xcode at or above the SDK minimum, patches re-applied. Keep a dated "current state" section and a history kept only where it explains the present. |

### 3. TypeScript — maximum strict

Baseline `tsconfig.json` for RN (Expo: `extends: "expo/tsconfig.base"`; bare: `extends: "@react-native/typescript-config"`), then override:

```jsonc
{
  "extends": "expo/tsconfig.base", // or "@react-native/typescript-config"
  "compilerOptions": {
    // MUST
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "noImplicitOverride": true,
    "noFallthroughCasesInSwitch": true,
    "noImplicitReturns": true,
    "forceConsistentCasingInFileNames": true,
    "isolatedModules": true,
    "skipLibCheck": true,
    "noEmit": true,
    "allowJs": false,
    "jsx": "react-jsx",
    "moduleResolution": "bundler",
    "resolveJsonModule": true,
    "esModuleInterop": true,
    "allowSyntheticDefaultImports": true,
    "verbatimModuleSyntax": true,

    // MUST — see §8 for aliases
    "baseUrl": ".",
    "paths": { "@/*": ["src/*"] },

    // SHOULD (stricter still; enable when the codebase can bear it)
    "noPropertyAccessFromIndexSignature": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "useUnknownInCatchVariables": true, // already implied by strict, listed for clarity

    // DECIDE and record — see the note below
    "exactOptionalPropertyTypes": false
  },
  "include": ["src", "*.ts", "*.tsx", ".expo/types/**/*.ts", "expo-env.d.ts"],
  "exclude": ["node_modules", "ios", "android", "coverage", "dist"]
}
```

The base configs set some of these and not others. `expo/tsconfig.base` (SDK 57) sets `module: preserve`, `moduleResolution: bundler`, `jsx`, `esModuleInterop`, `resolveJsonModule`, `skipLibCheck`, `noEmit` and **`allowJs: true`**, but **not** `strict`, `noUncheckedIndexedAccess` or `verbatimModuleSyntax`: a project that only extends the base is not strict. Keep the overrides so the intent is visible and survives a base-config change. Verify the base's current values in the audit (Phase 0) rather than assuming.

| Tier | Rule |
|---|---|
| MUST | All flags in the MUST block above. `strict: false` or any `strict*: false` override is a gap. |
| MUST | **`noUncheckedIndexedAccess` is on and respected.** Every `arr[i]`, `record[key]` and array destructuring is `T \| undefined`. Fix by intent, in this order: (1) static records with literal keys → `satisfies Record<K, V>` (keeps the literal keys, no `\| undefined`); (2) value can really be absent → a real guard (`if (!item) return`); (3) counter loops → `for…of`; (4) provably present → `?? default` / optional chaining; (5) `!` only as a last resort with a one-line comment saying why it is safe. Never `any` or `@ts-ignore`. DB getters that return `rows[0]` follow their sibling's contract (`rows[0] ?? null` for `T \| null`, or fold the absence into the existing row-count throw). This flag catches a real class of offline-first crashes: reading rows and keyed config maps for entries that may not exist. |
| MUST | No `// @ts-ignore`; `// @ts-expect-error` with a reason comment only. Enforce via `@typescript-eslint/ban-ts-comment`. If an existing project has it off, the count of `@ts-ignore` / `@ts-expect-error` is tracked in the migration plan and the rule is flipped to `warn` → `error` as the count falls. Review enforces it meanwhile and the docs say so. |
| MUST | `any` is forbidden in app code (`@typescript-eslint/no-explicit-any: error`); `unknown` + narrowing instead. `catch (error)` is `unknown`: use a shared `getErrorMessage(error)` helper, never `error.message`. Existing `any`s are tracked as debt in the migration plan, same ramp as above. |
| MUST | `npm run typecheck` = `tsc --noEmit` and is green on the main branch. The IDE's TypeScript LSP is a convenience; `tsc` is the ground truth. |
| SHOULD | **`exactOptionalPropertyTypes`: evaluate, decide, record.** It distinguishes "absent" from "set to `undefined`", which fights common React, Redux and UI-library prop idioms (`prop={cond ? value : undefined}`) and often produces hundreds of errors with little bug yield. Projects that turned it off deliberately write the decision next to the tsconfig notes so nobody re-litigates it. |
| SHOULD | Ambient types for assets (`.png`, `.svg`) and env in a single `src/types/` or `declarations.d.ts`; no scattered `declare module`. |
| SHOULD | Separate `tsconfig.build.json` / `tsconfig.test.json` only when needed (Jest types leaking into app code is the usual reason). |

### 4. ESLint — flat config (ESLint 9+)

Baseline `eslint.config.js`:

```js
// eslint.config.js
import js from "@eslint/js";
import tseslint from "typescript-eslint";
import react from "eslint-plugin-react";
import reactHooks from "eslint-plugin-react-hooks";
import reactNative from "eslint-plugin-react-native";
import importX from "eslint-plugin-import-x";
import eslintComments from "@eslint-community/eslint-plugin-eslint-comments";
import globals from "globals";
import prettier from "eslint-config-prettier"; // option A, see "Formatter wiring"

// typescript-eslint 8 deprecates its `tseslint.config()` helper in favour of `defineConfig` from "eslint/config"; both work.
export default tseslint.config(
  { ignores: ["node_modules", "ios", "android", ".expo", "coverage", "dist", "*.config.js"] },
  js.configs.recommended,
  ...tseslint.configs.strictTypeChecked,
  ...tseslint.configs.stylisticTypeChecked,
  {
    languageOptions: {
      parserOptions: { projectService: true, tsconfigRootDir: import.meta.dirname },
      globals: { ...globals.es2021, __DEV__: "readonly" },
    },
  },
  {
    files: ["**/*.{ts,tsx}"],
    plugins: {
      react,
      "react-hooks": reactHooks,
      "react-native": reactNative,
      "import-x": importX,
      "@eslint-community/eslint-comments": eslintComments,
    },
    settings: { react: { version: "detect" }, "import-x/resolver": { typescript: true } },
    rules: {
      ...react.configs.flat.recommended.rules,
      ...react.configs.flat["jsx-runtime"].rules,
      ...reactHooks.configs.recommended.rules, // v7+: includes the React Compiler rules
      "react-hooks/exhaustive-deps": "error",
      // React Compiler rules as errors (see rn-architecture-standards.md §11). Names per eslint-plugin-react-hooks v7:
      "react-hooks/set-state-in-effect": "error",
      "react-hooks/preserve-manual-memoization": "error",
      "react-hooks/refs": "error",
      "react-hooks/immutability": "error",
      "react-hooks/incompatible-library": "error",
      "react-hooks/rule-suppression": "warn", // not in `recommended`; enable explicitly. Only fires for suppressions the compiler detects

      // A suppressed react-hooks/* rule makes the compiler silently skip that component. Hard ban:
      "@eslint-community/eslint-comments/no-restricted-disable": ["error", "react-hooks/*"],
      "@eslint-community/eslint-comments/no-unlimited-disable": "error", // closes the bare `/* eslint-disable */` bypass
      "react-native/no-unused-styles": "error",
      "react-native/no-inline-styles": "warn",
      "react-native/no-raw-text": "error",
      "react/prop-types": "off",
      "@typescript-eslint/no-explicit-any": "error",
      "@typescript-eslint/ban-ts-comment": ["error", { "ts-expect-error": "allow-with-description" }],
      "@typescript-eslint/consistent-type-imports": ["error", { prefer: "type-imports", fixStyle: "separate-type-imports" }],
      "@typescript-eslint/no-floating-promises": "error",
      "@typescript-eslint/no-misused-promises": "error",
      "@typescript-eslint/switch-exhaustiveness-check": "error",
      "@typescript-eslint/no-unnecessary-condition": "error",
      "@typescript-eslint/no-shadow": "error",
      "import-x/order": ["error", {
        groups: ["builtin", "external", "internal", ["parent", "sibling", "index"], "type"],
        pathGroups: [{ pattern: "@/**", group: "internal" }],
        "newlines-between": "always",
        alphabetize: { order: "asc", caseInsensitive: true },
      }],
      "import-x/no-default-export": "error", // only with export style A, see §8
      "no-console": ["error", { allow: ["warn", "error"] }], // or allow all when a babel transform strips them, see runtime doc §B
      "no-restricted-imports": ["error", {
        paths: [
          { name: "react-redux", importNames: ["useDispatch", "useSelector"], message: "Use typed hooks from @/store/hooks" },
          { name: "react-native", importNames: ["SafeAreaView"], message: "Use react-native-safe-area-context" },
          // project-specific: retired packages, legacy APIs the project migrated away from, with the ticket in the message
        ],
      }],
      // Project-specific hazards as AST selectors — see "no-restricted-syntax" below.
      "no-restricted-syntax": ["error" /* , ...selectors */],
    },
  },
  {
    // Node tooling scripts (utils/*.mjs): node globals, not bundled into the app.
    files: ["utils/**/*.{js,mjs}", "scripts/**/*.{js,mjs}"],
    languageOptions: { globals: { ...globals.node, ...globals.es2021 } },
  },
  prettier, // option A only; always last
);
```

| Tier | Rule |
|---|---|
| MUST | Flat config (`eslint.config.js` / `.mjs`), ESLint ≥ 9, `typescript-eslint` ≥ 8 with **type-aware** rules (`projectService: true`). Legacy `.eslintrc.*` is a migration item. |
| MUST | `strictTypeChecked` base; `no-floating-promises`, `no-misused-promises`, `switch-exhaustiveness-check`, `exhaustive-deps` as errors. Typed linting roughly halves lint speed; that is the price, not a reason to skip it. |
| MUST | **Formatter wiring — detect, two options.** (A) `eslint-config-prettier` last in the chain and Prettier run separately (`format:check` in CI): ESLint never reports formatting. (B) `eslint-plugin-prettier` with `prettier/prettier: error`: formatting surfaces as lint errors and `lint --fix` formats; slower, one tool to run. Either is fine. The gap is having formatting rules in ESLint (`quotes`, `semi`, `max-len`) that *disagree* with Prettier — align them (`avoidEscape`, same width) or delete them. |
| MUST | `eslint-plugin-react-hooks` ≥ 7 with the React Compiler rules as errors, and `eslint-plugin-react-native` enabled. When the compiler is on (`rn-architecture-standards.md` §11), suppressing a `react-hooks/*` rule is itself a lint error (`no-restricted-disable` above); legacy suppressions live in one file-scoped override with a comment per file and may not grow. |
| MUST | `no-restricted-imports` for project-specific banned imports (typed Redux hooks, `SafeAreaView`, raw `fetch`, moment, retired packages after a migration — detect from project). Every entry's `message` names the replacement and the ticket. |
| MUST | **`no-restricted-syntax` for project-specific hazards.** When a bug class cannot be caught by an existing rule, write an AST selector with a message that points at the doc section. Typical uses: library APIs that break under the React Compiler (render-time subscription reads, see architecture §11), options removed by a navigation major upgrade (`unmountOnBlur`, `headerBackTitleVisible`, …), `beforeRemove` where the project has a back-guard hook, web-only branches (`Platform.OS === 'web'`, `Platform.select({ web })`) in a native-only app. A recurring trap gets a selector, not a review comment. |
| MUST | `npm run lint` = `eslint <src dirs>` with `--max-warnings 0` in CI/hooks. |
| SHOULD | `eslint-plugin-import-x` with `order` + resolver; `eslint-plugin-redux-saga` if sagas are used; `eslint-plugin-jest` / `testing-library` for test files. |
| SHOULD | Expo projects: start from `eslint-config-expo` flat preset and layer the above on top rather than replacing it. |
| SHOULD | `@typescript-eslint/no-magic-numbers` (ignore `-1, 0, 1`, defaults, enums, literal types) pushes durations and limits into named `UPPER_SNAKE_CASE` constants. `complexity: warn` flags the generators and components that need splitting. |
| SHOULD | Hardening candidates are listed in the docs with the reason each is not on yet, so "stricter lint" is a backlog, not a vague wish. |

### 5. Prettier & EditorConfig

```jsonc
// .prettierrc
{
  "semi": true,
  "singleQuote": true,
  "trailingComma": "all",
  "printWidth": 100,
  "tabWidth": 2,
  "arrowParens": "always",
  "endOfLine": "lf",
  "plugins": [] // add prettier-plugin-tailwindcss if NativeWind
}
```

```ini
# .editorconfig
root = true
[*]
charset = utf-8
end_of_line = lf
insert_final_newline = true
trim_trailing_whitespace = true
indent_style = space
indent_size = 2
[*.md]
trim_trailing_whitespace = false
```

| Tier | Rule |
|---|---|
| MUST | Prettier config committed; `.prettierignore` mirrors ESLint ignores plus lockfile, `ios/`, `android/`. |
| MUST | **Documentation is formatted too.** The `format:check` / `format` scripts include the Markdown docs (`README.md`, the docs tree, agent-instruction files, slash-command files), and `lint-staged` formats staged `*.md`. Unformatted tables and wrapped lines are the first thing to rot in a docs tree. |
| MUST | `endOfLine: "lf"` + `.gitattributes` with `* text=auto eol=lf` (mixed-OS teams; Windows CRLF breaks Gradle scripts and shell hooks). |
| MUST | `.editorconfig` committed. |
| SHOULD | Specific values above are a default; **detect** and keep the project's existing style (a 140 `printWidth` with `trailingComma: es5` is as valid as the baseline) — consistency beats the particular choice. Only `endOfLine` and a consistent `trailingComma` are worth fighting for. |
| SHOULD | `.vscode/settings.json` + `extensions.json` committed with format-on-save, ESLint fix-on-save, recommended extensions (ESLint, Prettier, Expo Tools). |

### 6. Git hooks — husky + lint-staged + commit-msg

```jsonc
// package.json (excerpt)
{
  "scripts": {
    "prepare": "husky",
    "typecheck": "tsc --noEmit",
    "lint": "eslint src/ --max-warnings 0",
    "format:check": "prettier --check ./src ./docs \"./*.md\"",
    "validate": "npm run typecheck && npm run lint && npm run format:check && npm run test:ci"
  },
  "lint-staged": {
    "src/**/*.{ts,tsx}": ["eslint --fix --max-warnings 0", "prettier --write"],
    "*.{js,mjs,json,md,yml,yaml}": ["prettier --write"]
  }
}
```

```sh
# .husky/pre-commit — repo invariants first (each < a few seconds), then lint-staged
npm run compare-locales      # locale files have the same keys (runtime doc §D)
npm run check-module-map     # every src/ folder has a module-map entry (docs doc §2)
npm run check-doc-links      # no doc points to a missing file, heading or source path (docs doc §3)
npx lint-staged
# .husky/commit-msg — option A
npx --no -- commitlint --edit "$1"
# .husky/pre-push
npm run typecheck && npm run test:ci   # drop test:ci at testing Level 0
```

| Tier | Rule |
|---|---|
| MUST | husky installed via `prepare`; `pre-commit` runs lint-staged (lint + format on staged files only); `commit-msg` validates the header (§7). |
| MUST | lint-staged runs ESLint with `--max-warnings 0`; a warning is a failed commit. |
| MUST | **Repo-invariant checks run in `pre-commit`.** Any rule of the form "X and Y must agree" (locale files, module map vs `src/` folders, docs vs files they name, a coverage list vs the tables it should cover) is a small Node script with a header comment, exit code 1 and a clear message, wired into the hook. Each must finish in a few seconds; slower checks go to pre-push or CI. A documented invariant without a script rots within weeks. |
| SHOULD | `pre-push` runs typecheck (+ tests when a runner exists). Full typecheck on pre-commit is too slow on large RN apps. |
| SHOULD | Hooks are bypassable with `--no-verify` but the README states it is for emergencies only; CI (if present) re-runs everything. |

### 7. Conventional commits & automated releases

Two accepted commit-header conventions — **detect**, keep the one in use, document it in the agent-instructions file:

**Option A — scoped:** `type(scope): subject`, scopes enumerated from feature folders.

```js
// commitlint.config.js
export default {
  extends: ["@commitlint/config-conventional"],
  rules: {
    "scope-enum": [2, "always", ["app", "auth", "todos", "store", "ui", "nav", "deps", "ci", "release"]], // detect from repo structure
    "subject-case": [2, "never", ["sentence-case", "start-case", "pascal-case", "upper-case"]],
    "body-max-line-length": [0],
  },
};
```

**Option B — work-item:** `type: #<id> - <summary>` where `#<id>` is the issue-tracker item (Azure Boards, Jira, GitHub). The summary describes the resulting behaviour; details go in the body. The release tool links the id (`issueUrlFormat`). Enforced by commitlint with a custom `header-pattern`, or by a shell regex in `.husky/commit-msg` that also accepts git's default `Merge …` / `Revert …` headers and caps the header length:

```sh
# .husky/commit-msg — option B, no commitlint dependency
if ! head -1 "$1" | grep -qE "^(Merge[: ].{1,}|Revert[: ].{1,}|(feat|fix|chore|docs|test|style|refactor|perf|build|ci|revert)(\(.+?\))?: .{1,})$"; then
  echo "Expected: <type>(<scope>)?: <subject>, type ∈ feat|fix|chore|docs|test|style|refactor|perf|build|ci|revert" >&2; exit 1
fi
head -1 "$1" | grep -qE "^.{1,200}$" || { echo "Header longer than 200 characters" >&2; exit 1; }
```

| Tier | Rule |
|---|---|
| MUST | Conventional types enforced by the `commit-msg` hook: `feat`, `fix`, `perf`, `refactor`, `docs`, `test`, `style`, `build`, `ci`, `chore`, `revert`. `BREAKING CHANGE:` footer or `!` for majors. Header length capped (72–200, project's choice). |
| MUST | A release tool driven by the commits. **Detect** and keep one of: `semantic-release`, `release-please`, `commit-and-tag-version` (standard-version). If none: recommend `release-please` when CI exists (review PR, no direct pushes), `commit-and-tag-version` when releases are cut locally (`rn-cicd-standards.md` §0 profile B). |
| MUST | `CHANGELOG.md` generated by the tool, never hand-edited; version in package.json is bumped only by the tool. With `commit-and-tag-version`: `types` → changelog sections, `commitUrlFormat` / `compareUrlFormat` / `issueUrlFormat` pointed at the repo host (GitHub, Azure DevOps, GitLab), a `postchangelog` script may strip the release's own `chore(release)` entries. |
| MUST | **Tool attribution is a team decision, written down.** Whether AI assistants may add `Co-Authored-By` or "generated with" trailers is decided once and stated in the agent-instructions file; agents follow it. Silence means the trailers appear in the changelog. |
| SHOULD | Option A: scopes enumerated (`scope-enum`), kept in sync when features are added. Option B: the header is also the search key into docs and history, so the id is mandatory, and code comments that exist because of a ticket carry the same `#<id>`. |
| SHOULD | Squash-merge PRs with the PR title as the conventional commit, so the history that feeds the changelog stays clean. |
| SHOULD | Commit message template in `.gitmessage` or a commitizen prompt (`cz-git`) to lower the friction. |

### 8. Path aliases & import order

| Tier | Rule |
|---|---|
| MUST | One root alias `@/` → `src/` (an extra `@assets/` → `assets/` is fine) configured in **all** places that must agree: `tsconfig.paths`, the bundler (Expo: `experiments.tsconfigPaths` / Metro `resolver`; bare: `babel-plugin-module-resolver`), and Jest `moduleNameMapper` when Jest exists. A mismatch between these is a classic "works in editor, fails in app" bug. |
| MUST | No relative imports that climb more than one level (`../../..`); enforce with `no-restricted-imports` patterns or `import-x/no-relative-parent-imports`. |
| MUST | Import order enforced by `import-x/order` (builtin → external → internal `@/` → relative → type), alphabetized, auto-fixed by lint-staged. |
| MUST | **Export style — detect, two options**, applied consistently. (A) Named exports only (`import-x/no-default-export`), exceptions documented: route files for file-based routers, config files, `App.tsx`. (B) One default export per component file, `export default React.memo(Component)` at the bottom, file name = component name; non-component modules (slices, utils, hooks) use named exports. Option B pairs with the "one component per file" rule (`rn-architecture-standards.md` §1) and makes the file name the import name; it does not use `no-default-export`. |
| SHOULD | Feature folders expose an `index.ts` barrel only for their public API; deep imports into another feature's internals are banned (`import-x/no-internal-modules` or `eslint-plugin-boundaries`). Flat feature-module layouts (architecture §1 option B) may skip barrels and document the allowed cross-module imports instead. |
| SHOULD | `type` imports via `consistent-type-imports` (`fixStyle: separate-type-imports`) so Babel/Metro can drop them. |

### 9. Env, environments & secrets

Two accepted **environment models — detect**:

- **(A) Build-time env**: `EXPO_PUBLIC_*` / `react-native-config` values per build profile, read through one typed module.
- **(B) Runtime environment selector**: one binary serves every environment and tenant; the backend address, tenant/company and login options arrive at first launch (QR code, deep link, or a code the user types), ideally as an encrypted payload the app validates, and are persisted. Server-side configuration (`mobile/config`-style endpoint) then drives features (runtime doc §E). The bundle contains no backend URL at all. Suits multi-tenant field apps with dev/UAT/prod backends chosen per device.

| Tier | Rule |
|---|---|
| MUST | No secrets in the repo: `.env*` (except `.env.example`) in `.gitignore`, `.env.local` included; `.env.example` committed with every key and a dummy value. |
| MUST | Model A: a single typed access point (`src/config/env.ts`) that reads the raw env, validates with zod (or equivalent) at startup, and exports a typed, frozen object. No `process.env.X` anywhere else. Model B: a single slice + selector pair holds the selected environment; the payload is validated (tenant name must be a known enum value) before it is stored; a dev-only switcher screen can change tenant and environment without a new code. |
| MUST | Public vs private distinction is documented and enforced: anything bundled into the app is public (`EXPO_PUBLIC_` prefix makes this explicit). API keys that must stay secret cannot live in the app at all — they belong behind a backend. |
| MUST | Model A: per-environment config (dev / staging / prod) selected by build profile, not by editing files (`eas.json` env per profile, or `.env.staging` + schemes/flavors). Model B: "change environment" is a full local wipe path (databases, persisted state, sync timestamp) — see `redux-saga-best-practices.md` §9 profile C. |
| SHOULD | A secret-scanning hook (`gitleaks` via pre-commit or CI) and `npm audit` policy (fail on high/critical in CI, documented exceptions). |
| SHOULD | Signing keys, keystores, service-account JSON stored only in EAS secrets / CI secret store / vault; `*.keystore`, `*.p12`, `*.mobileprovision`, `*.p8` git-ignored. `google-services.json` / `GoogleService-Info.plist` are committed only by explicit decision (they are public identifiers, not secrets, but the decision is written down). A CI step that writes a secret into a committed config file by string replacement is a finding (`rn-cicd-standards.md` §2). |

### 10. RN build & release

| Tier | Rule |
|---|---|
| MUST | App version (`version` in package.json / app.json) is the semver produced by the release tool. Native build numbers (`ios.buildNumber`, `android.versionCode`) are **monotonic and never hand-edited**: EAS `autoIncrement` (remote app version source), Fastlane `increment_build_number` in CI, or a build script that stamps them from the release version/date before `prebuild` (`rn-cicd-standards.md` §3). Placeholder values in the committed `app.json` are acceptable only when the stamping script is the only way a build is produced. |
| MUST | Build profiles defined in config (Expo: `eas.json` with `development`, `preview`, `production`, plus variants such as `production-apk` that `extends` production; bare: Fastlane lanes + schemes/flavors) with env per profile (§9). |
| MUST | **Continuous Native Generation discipline (Expo prebuild):** `ios/` and `android/` are git-ignored and regenerated; every native change goes through `app.json` / `app.config.*`, a config plugin (`plugins/`) or `expo-build-properties`, never by editing the generated folders. The README says when the dev client must be rebuilt (native dependency, `app.json`, config plugin, a file in `patches/`). |
| MUST | **No debug-signed deliverables.** Local `expo run:android` / Gradle builds — the `release` variant included — are signed with the public debug keystore; anyone can forge such a build and server-side certificate checks (Firebase auth, Play Integrity) reject it. Binaries handed to customers come only from a scripted path (`eas build` with managed credentials, or Fastlane with `match`) that ends in a **signature verification script** failing on debug certificates (`apksigner verify --print-certs` / `keytool -printcert -jarfile`, optional expected SHA-256 pin). See `rn-cicd-standards.md` §3. |
| MUST | Reproducible native builds: `npm ci`, pinned Node (§1), pinned Ruby/CocoaPods version (`.ruby-version`, `Gemfile.lock`) for iOS, pinned JDK/Gradle (`gradle-wrapper.properties`, document JDK version) — or EAS, where `eas.json` pins the image. |
| MUST | OTA updates (EAS Update) are tied to a runtime version policy (`fingerprint` or `appVersion`) so JS updates never reach incompatible native builds. Projects that do not ship OTA say so in `docs/release.md`. |
| SHOULD | Release checklist documented in `docs/release.md`: branch, release tool run, native build, store submission, OTA. For a tenant that gets its own build (`rn-cicd-standards.md` §3b), a per-tenant checklist. |
| SHOULD | Source maps uploaded to the crash reporter (Sentry) per release; release name = app version + build number (runtime doc §B). Local verification builds set the uploader's disable flag (`SENTRY_DISABLE_AUTO_UPLOAD=true`) so they neither fail on missing tokens nor pollute the release list. |
| SHOULD | Expo: `expo-doctor` in `validate`; bare: `npx react-native doctor`. Both in CI if present. |
| SHOULD | `app.json` / `app.config.ts` is TypeScript (`app.config.ts`) when it depends on env; keeps config typed and testable. With a config-stamping script (§10 first row) `app.json` stays JSON because the script rewrites it. |

### 11. Repository hygiene & other

| Tier | Rule |
|---|---|
| MUST | `.gitignore` covers: `node_modules`, `.expo`, `ios/` and `android/` under CNG (or `ios/Pods`, `ios/build`, `android/build`, `android/.gradle` when the native folders are committed), `*.keystore`, `.env*`, `coverage`, `*.log`, `.DS_Store`, IDE folders except the committed `.vscode` files, extracted databases and other local artifacts (see the scratch-folder rule). |
| MUST | README is the **local development** guide and nothing else: requirements (Node version and why, Xcode/Android Studio limits), install command, how to run on each platform, where the environment comes from, what to run before committing, and links to the docs tree for everything about the code. Knowledge about the code lives in the docs tree (`rn-docs-and-agents-standards.md`), not in a growing README. |
| MUST | A verification baseline per `rn-testing-standards.md` §0: either a test runner with coverage thresholds, or a declared Level 0 (static checks + debug tooling + device verification) stated in the docs so nobody assumes a runner exists. |
| MUST | A gitignored **scratch folder** (`temp-local/`) for extracted databases, screenshots, exports, local build outputs and notes. Scripts write there by default; doc link checkers skip it; nothing project-relevant lives only there. |
| SHOULD | `CODEOWNERS` and a PR template with a checklist (tests or verification steps, screenshots for UI, changelog impact). |
| SHOULD | Dependency hygiene: `npx depcheck` / `knip` run occasionally; unused deps removed; `npm ls` clean (no unmet peers). |
| SHOULD | Agent instructions file (`CLAUDE.md` / `AGENTS.md`) with the MUST rules as bullets and the scripts to run, so coding assistants follow the same standards. Shape and maintenance rules: `rn-docs-and-agents-standards.md`. |
| SHOULD | Generated reference files (SQLite DDL dumps, API type outputs) are regenerated by a script and never hand-edited; the file header says which script. |

---

## Part 2 — Audit prompt (for Claude Code / agentic tools)

```text
<<<<< PROMPT START >>>>>

You are auditing this React Native repository's tooling and project standards against "the rules" in `rn-project-standards.md` (Part 1). Read that file first; if it is missing, ask me for it and stop.

Phases run in order. Do not modify anything except documentation and the migration plan file until I approve in Phase 5.

## Phase 0 — Inventory
Collect and report, with file paths:
- RN flavor: Expo (SDK version, managed vs prebuild / dev client, Expo Router?) or bare (RN version). Monorepo? Workspaces? Are `ios/` and `android/` committed or generated (CNG)?
- Package manager and lockfile actually in use (if not npm, say so and ask whether to keep it; the rules are written for npm but every rule has an equivalent).
- Node pinning method present (.nvmrc, .node-version, volta, .tool-versions, engines, packageManager, engine-strict). Whether `.npmrc` sets `legacy-peer-deps`.
- Dependency policy: does `npx expo-doctor` pass? Does `npx expo install --check` report drift? Which packages are patched (`patches/`), is there a regen script, do patch files contain build artifacts (`grep "^diff" patches/*.patch`)?
- tsconfig(s) and every strictness flag's current value, including what the extended base sets. Is `exactOptionalPropertyTypes` decided and documented?
- ESLint: config format (flat vs legacy), ESLint and typescript-eslint versions, plugins and presets, whether type-aware rules are on, formatter wiring (option A `eslint-config-prettier` / option B `eslint-plugin-prettier` / neither / both), `eslint-plugin-react-hooks` version and whether the React Compiler rules are errors, whether suppressing `react-hooks/*` is banned, existing `no-restricted-syntax` / `no-restricted-imports` entries, which folders `npm run lint` covers, whether Node scripts have a globals block.
- Prettier, EditorConfig, .gitattributes, .vscode; whether `format:check` covers the Markdown docs.
- Git hooks: husky/lefthook/simple-git-hooks, lint-staged, commit-msg mechanism (commitlint or shell regex); what each hook runs; which repo-invariant scripts exist (locale parity, module map, doc links, others) and whether they are wired.
- Commit convention in use: sample the last 50 commits, classify as option A (scoped), option B (work-item `#id`), or neither, and report the % that parse.
- Release tooling and CHANGELOG state; how version/buildNumber/versionCode are managed (release tool, EAS remote, stamping script, hand-edited?); whether tool-attribution trailers appear in history and whether a policy is documented.
- Aliases: tsconfig paths vs babel/metro vs jest moduleNameMapper (if Jest exists) — do they agree? Export style in use (count default vs named exports of components).
- Environment model: A (build-time env) or B (runtime selector); how env is read, whether a typed/validated access point exists, what is git-ignored, whether any secret-looking values are committed (run a quick scan for API keys, tokens, keystores, `.p8`, service-account JSON).
- Build/release: eas.json / Fastlane, profiles, OTA runtime policy or "no OTA", Ruby/CocoaPods/JDK pinning, deliverable build scripts, signature verification script, config-stamping script for multi-tenant builds.
- Verification baseline: test runner present (preset, RNTL, coverage) or Level 0 declared in docs; is the status stated anywhere?
- Scratch folder (`temp-local/` or similar) and its gitignore status; generated reference files and their regeneration scripts.
- Existing docs and agent instruction files (README, docs/ or another docs tree, CONTRIBUTING, CLAUDE.md, AGENTS.md, GEMINI.md, .cursorrules, slash-command files) and their style. Note where project knowledge actually lives — do not assume `docs/`.
Run `npm run typecheck`, `npm run lint`, `npm run format:check` (or the project's names) and `npm test` only if a test script exists; report pass/fail with counts (errors, warnings, `any` count, `@ts-ignore` count, `@ts-expect-error` count, react-hooks suppressions).

## Phase 1 — Confirm detect-rules with me
For every rule marked "detect", state what you found and what you intend to keep or add:
- Node pinning method
- Package manager
- Check-script names (canonical or the project's existing spelling)
- Release tool (or your recommendation if none, with reason) and delivery profile (CI-driven or local release, see `rn-cicd-standards.md` §0)
- Commit convention (option A scoped / option B work-item) and the tool-attribution policy
- Formatter wiring (option A / option B)
- Export style (option A named / option B default-per-component)
- Prettier style (keep existing or adopt baseline)
- Environment model (A build-time / B runtime selector)
- `exactOptionalPropertyTypes` decision
- Native config files' git-ignore status (google-services.json etc.)
Ask me to confirm or override these in one message. Wait for the answer.

## Phase 2 — Audit
For every rule in Part 1 (keep §numbering): status ✅ / ⚠️ / ❌ / ➖ (n/a with reason), evidence, tier, fix size S/M/L/XL, approximate files touched, risk low/medium/high.
Sizing: S = under 1h, mechanical config; M = half a day, touches many files mechanically (e.g. import order autofix, alias rewrite); L = a few days, requires code changes to satisfy (e.g. strict TS flags producing hundreds of errors, flat-config migration with custom rules); XL = a week+, (e.g. removing all `any`, `noUncheckedIndexedAccess` on a large codebase).
For TypeScript strictness: enable each missing flag one at a time in a scratch run and report the **error count per flag** — that is the real size.
For ESLint: run the baseline config in a scratch file and report the **error count per rule**.
Never claim compliance without evidence.

## Phase 3 — Gap report (chat)
1. Table: rule → tier → status → size → files → risk; MUST first.
2. Totals: MUST gaps, SHOULD gaps; total effort for "all MUST" and "all MUST + SHOULD".
3. Top 5 highest-value fixes with one-line justification. Typically: engine-strict + lockfile, repo-invariant checks in pre-commit, alias agreement, secrets scan results, type-aware lint, react-hooks suppression ban when the compiler is on.
4. Anything that should become a documented accepted exception rather than a fix.
5. Any **security findings** (committed secrets, keystores, debug-signed deliverable paths) go at the top of the report regardless of size, with a note that rotating the secret is required even after removing it from git.

## Phase 4 — Documentation and migration plan (write now)
1. Agent instructions file (CLAUDE.md / AGENTS.md / existing equivalent; create CLAUDE.md if none): add or extend a "Project standards" section — the scripts to run before committing, MUST rules as short bullets, alias convention, env model, commit format and attribution policy, release command, dependency policy in one line. Keep it short; detail goes to the rule file below.
2. Human docs: extend the project's existing docs location (detected in Phase 0; `docs/` only if nothing exists), or create `project-standards.md` (full rules adapted to this project's real paths and tools), `release.md` (the release checklist) and, if missing, a scripts guide. Update README's setup section (Node version + reason, `npm ci`, env setup, when to rebuild the dev client) if it is missing or wrong.
3. Match existing tone/style; reference rather than duplicate anything already documented.
4. Create or update `project-standards-migration-plan.md` in the docs location: date, confirmed detect-decisions from Phase 1, full gap table with a checkbox per item (`- [ ] §3 noUncheckedIndexedAccess — L, 142 errors across 38 files, low risk`), suggested order (security → MUST/S → MUST/M → MUST/L-XL as separate PRs → SHOULD), accepted exceptions.
Show a diff summary of what you wrote.

## Phase 5 — Ask before applying
Offer, with sizes and risk: "everything", "all MUST", "pick by §number", and your recommendation. Default recommendation:
- Now, in one PR: any security fix, §1 pinning, §2 scripts and dependency-policy doc, §5 Prettier/EditorConfig/gitattributes, §6 hooks and invariant scripts, §7 commit hook — these are config-only and low risk.
- Separate PR each: §4 flat-config migration or rule hardening, §8 alias unification + import-order autofix (large mechanical diff), §9 typed env module or selector validation.
- Separate PR per flag, in order of lowest error count: §3 strict flags; `any` / `@ts-ignore` removal as ongoing debt with the lint rule set to `warn` → `error` once the count hits zero.
- §10 release/build changes only with someone who can test a real build.
Wait for my answer; ask again if ambiguous.

## Phase 6 — Apply what I approved
- One §item per commit, conventional commit message in the project's convention, tick the migration plan as you go.
- After each item run the relevant check (`npm run typecheck`, `npm run lint`, `npm run format:check`, `npm test` only if it exists, `npx expo-doctor` / `npx react-native doctor` where relevant) and show the result.
- Mechanical rewrites (import order, alias) via the tool's autofix, never by hand-editing files one at a time.
- For TS flags: enable the flag, fix errors with real types (no `any`, no `!` non-null assertions as a blanket fix, no `@ts-expect-error` sprinkling); if the count is too large, stop and propose a per-folder plan.
- Never commit, print, or move secrets; if a secret is found, remove it from the working tree, add it to .env.example as a placeholder, and tell me to rotate it.
- If something is larger or riskier than estimated, stop and ask.
Finish with: what changed, what remains, updated totals in the migration plan.

<<<<< PROMPT END >>>>>
```

### Adapting

- **Bare RN without Expo:** the LLM will skip EAS and CNG items and use Fastlane/scheme equivalents; nothing to change in the prompt.
- **pnpm/yarn project:** tell it "keep pnpm" at Phase 1; `engine-strict` becomes `engineStrict` in `.npmrc` (pnpm) or `engines` + Corepack (yarn).
- **Azure DevOps / GitLab:** the work-item commit convention (option B) and the release tool's URL formats point at that host; see `rn-cicd-standards.md` for the pipeline side.
- **Re-run:** point it at the existing migration plan and ask for a status refresh only.
