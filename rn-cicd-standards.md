# React Native CI/CD & Delivery Standards + Audit Prompt

> Standards version: 2 (2026-10-05). What changed: see `STANDARDS-INDEX.md` → Version history.

Part 1: rules (MUST / SHOULD). Part 2: audit prompt for Claude Code. Companion to the other `*-standards.md` files (this one assumes `rn-project-standards.md` §1, §6, §7, §10 — pinning, hooks, conventional commits, build profiles).

CI platform is **detected** (GitHub Actions, Azure Pipelines, GitLab CI, Bitrise, CircleCI, EAS Workflows…); examples use GitHub Actions with Azure Pipelines equivalents where they differ. Build system is **detected** (EAS for Expo, Fastlane for bare). Maestro is the assumed E2E runner (`rn-testing-standards.md` §8), when the project is at a level that has E2E.

---

## Part 1 — Rules

### 0. Delivery profile — declare one [detect]

| Profile | What it is | Who it fits |
|---|---|---|
| **A — CI-driven** | Three pipelines (validate / preview / release) on a CI platform; production builds and store submission only from the release pipeline. §1–§8 apply in full. | Teams with a maintained CI and a release cadence that justifies it. |
| **B — Local release** | No runnable CI. Releases are cut on a developer machine: the release tool bumps and tags, `eas build --local` (or Fastlane) produces signed binaries from scripted entry points, a signature check runs, and a checklist drives store/tenant hand-over. §1–§2 apply as "what local must guarantee instead"; §3–§5 apply with the local equivalents below; §6–§8 are deferred and said so. | Small teams, one or two releases a month, multi-tenant deliverables handed over outside the stores. |

| Tier | Rule |
|---|---|
| MUST | The profile is stated in `release.md` and the agent-instructions file. A repo with a pipeline file that is **disabled, stale or half-commented** (an old Node version, `npm install`, steps with `enabled: false`, secrets written into files) is **profile B with a trap**: either delete the file or put a header comment on it saying it is not the release path and what is. An agent that reads it as the release process will follow it. |
| MUST [B] | **Local release discipline** replaces the pipeline: (1) release only from the main branch at a clean, pushed state, (2) `npm run validate` (or the project's check scripts) green, (3) the release tool bumps version + CHANGELOG + tag in one commit, (4) every deliverable binary comes from **one npm script per deliverable** (`<platform>:build:eas-local[:<tenant>[-apk]]`) that applies the tenant config (§3b), builds with the managed credentials, writes to the scratch folder and ends in the signature check (§3), (5) the checklist in `release.md` is ticked, including the store or tenant hand-over steps and who confirms installation. |
| MUST [B] | What CI would have guaranteed is guaranteed locally and written down: pinned Node (`engine-strict`), `npm ci` for a release build (or a clean `node_modules`), `expo-doctor` clean, local Xcode at or above the SDK minimum, EAS CLI version pinned in `eas.json` (`cli.version`). |
| SHOULD [B] | The trigger for moving to profile A is written down (team size, release frequency, a second platform target), and the first pipeline to add is `validate`, not builds. |

### 1. Pipeline shape (profile A)

```
PR opened / updated                 merge to main                tag vX.Y.Z (release tool)
──────────────────────              ─────────────────            ────────────────────────
validate (fast, <10 min)            validate                     build: production (iOS+Android, per tenant)
  install (npm ci, cached)          build: preview (internal)    verify signatures
  typecheck                         e2e smoke on preview         e2e smoke on the artifact
  lint --max-warnings 0             ota: publish to "preview"    submit to TestFlight / Play internal
  format:check                                                   ota channel: production (after store approval)
  repo-invariant scripts                                         sentry: upload source maps, create release
  test:ci (+ coverage upload)  [testing Level ≥ 1]               tenant deliverables to the hand-over location
  commitlint (PR title / commits)
  expo-doctor / rn doctor
  (optional) build: dev-client when native deps changed
```

| Tier | Rule |
|---|---|
| MUST | Three pipelines exist and are distinct: **validate** (every PR), **preview/staging** (merge to main), **release** (tag or release PR). No pipeline does "everything on every push". |
| MUST | `validate` runs exactly the `npm run validate` script from `rn-project-standards.md` §2 (typecheck, lint, format check, repo-invariant scripts, `test:ci` when a runner exists) plus commitlint; CI and local hooks call the same scripts so "passes locally, fails in CI" is only ever environment, not config. |
| MUST | Required status checks on `main`: validate must pass; branch protection (GitHub) / branch policies (Azure DevOps) forbid direct pushes and force pushes. |
| MUST | `validate` finishes in under 10 minutes; native builds are **never** part of PR validation by default (they run on main / release or on a label like `build-needed`). |
| SHOULD | Native build on PR only when native-affecting files change (`package.json` deps, `app.config.*`, `plugins/`, `patches/`, `*.podspec`, `ios/`/`android/` when committed): use path filters or an Expo fingerprint check (`@expo/fingerprint`) to decide. |
| SHOULD | Concurrency groups cancel superseded runs on the same PR. |

### 2. Environment & reproducibility in CI

| Tier | Rule |
|---|---|
| MUST | Node version in CI comes from the repo's pin file (`node-version-file: .nvmrc` / volta / `.tool-versions`; Azure: `UseNode@1` with `versionSpec` read from the file in a prior step), never hardcoded in the workflow. A pipeline on a Node major the SDK no longer supports is a finding even when disabled. |
| MUST | `npm ci` with the lockfile; cache keyed on `package-lock.json` hash. Never `npm install` in CI. |
| MUST | iOS: Ruby from `.ruby-version`, `bundle install` from `Gemfile.lock`, CocoaPods cached on `Podfile.lock`. Android: JDK version pinned in the workflow and documented; Gradle cache enabled. (Or delegated entirely to EAS, in which case `eas.json` pins the image/`node`/`bun`/`xcode` versions.) |
| MUST | Secrets only via the CI secret store (GitHub secrets / Azure variable groups marked secret) or EAS secrets. A workflow that `echo`es a secret, **writes it into a committed config file by string replacement** (`Set-Content` / `sed` of an `INSERTKEYHERE` placeholder), or passes it as a plain-text arg is a finding: the placeholder file is committed, the written file is one `git add -A` away from leaking, and the step cannot be reviewed. Write secrets to a path under the runner's temp folder that is listed in `.gitignore`, or use the build tool's native secret file support (`eas secret:create --type file`). |
| MUST | Fork PRs run `validate` without secrets; jobs that need secrets (builds, OTA, submit) require the PR to be from the repo or a maintainer label. |
| SHOULD | Pinned action versions by SHA (or at minimum major tag) for third-party actions; Dependabot/Renovate keeps them updated. Azure: pinned task major versions (`Npm@1`, `PowerShell@2`) and a self-hosted agent pool whose tool versions are documented. |
| SHOULD | Timeouts on every job; no job may run unbounded. |

### 3. Builds — EAS (Expo) / Fastlane (bare) [detect]

| Tier | Rule |
|---|---|
| MUST | Build profiles map 1:1 to environments: `development` (dev client), `preview` (internal distribution, staging env), `production` (store, prod env), plus deliverable variants that `extends` production (`production-apk` with `buildType: apk` for tenants who sideload). Env vars per profile per `rn-project-standards.md` §9/§10; runtime-environment-selector apps (model B) need none. |
| MUST | **Build numbers are monotonic and never hand-edited.** Accepted mechanisms, one per project: EAS `autoIncrement` with `cli.appVersionSource: "remote"`; Fastlane `increment_build_number` from the CI run number; or a **config-stamping script** run before `prebuild` that writes `expo.version` from the release tool's version and `ios.buildNumber` / `android.versionCode` from a monotonic formula (date-based `yyyyMMddHH`, or the Play/App Store latest + 1), with placeholder values in the committed `app.json`. The script is the only way a build is produced (it is called from the build scripts), and `--skip-versioning` exists for local verification builds. |
| MUST | Production builds are triggered only from the release pipeline (profile A) or from the documented local release scripts on the main branch (profile B); never from an unrelated branch or an `expo run:<platform>` build. |
| MUST | **Signing is explicit and verified.** Local `expo run:android` / Gradle builds sign **every** variant, `release` included, with the public debug keystore (`android/app/debug.keystore`) because the generated `build.gradle` routes `release` through `signingConfigs.debug`. Such a build must never reach a user: it is forgeable, and server-side certificate checks (Firebase Auth `auth/invalid-cert-hash`, Play Integrity) reject it. Every deliverable path ends in `npm run <platform>:verify-signature -- <file>` (`apksigner verify --print-certs` / `keytool -printcert -jarfile`, fails on the debug certificate's known fingerprints, optional `--expect-sha256` pin). The debug keystore's fingerprint is **never** registered with Firebase or any attestation service. |
| MUST | **Signing keys are inventoried** in the docs: which key signs which application id on which path (EAS keystore per application id, a Play-created key for store builds, the debug keystore), where each lives, which are registered with Firebase / attestation, and whether two paths sign the same app id differently (a user base that switches keys must **uninstall**, which deletes unsynced local data — plan a "sync everything first" step). |
| MUST | Every build is traceable: the build has the git SHA, the app version, the build number, and the CI run URL or local build log (EAS metadata or Fastlane `lane_context`), surfaced in a "debug info" screen in non-production builds. |
| SHOULD | Credentials managed by EAS (`eas credentials`) or `fastlane match` with a private git repo; no `.p12`/`.keystore` in CI variables as base64 blobs unless `match` is impossible — document the decision. |
| SHOULD | Build cache (EAS cache / Fastlane + actions cache) enabled; cold build time documented. Local verification builds (`xcodebuild build … CODE_SIGNING_ALLOWED=NO`, `expo run:ios --configuration Release`) set `SENTRY_DISABLE_AUTO_UPLOAD=true`. |
| SHOULD | Bare RN: Fastlane lanes `build_ios`, `build_android`, `beta`, `release` are the only entry points; CI calls lanes, never raw `xcodebuild`/`gradlew`. |

### 3b. Multi-tenant (white-label) builds [detect]

Applies when one code base ships under more than one store identity (a tenant with its own MDM store that rejects a package name already on Google Play, a partner brand). This is a **build-time** axis, separate from any **runtime** tenant selection the app does (`rn-project-standards.md` §9 model B): a tenant build still needs that tenant's runtime configuration.

| Tier | Rule |
|---|---|
| MUST | Tenant build configs live in **one module** (`utils/company-build-configs.mjs` style): a `DEFAULT` config and per-tenant entries that inherit it and override only what differs (display name, URL scheme, iOS `bundleIdentifier`, Android `package`, Firebase file paths, optional static build numbers). Tenant keys are build identities, not the runtime tenant enum; the docs say so. |
| MUST | One **apply script** (`ci:apply-company-config -- --company=<key> [--platform] [--skip-versioning]`) rewrites `app.json` in place from the config module and stamps versions (§3); every tenant build script calls it first; nobody edits `app.json` by hand for a tenant build. The default config is applied back (or `git checkout app.json`) after a local tenant build so the working tree is clean. |
| MUST | **One keystore / signing identity per application id**, managed by the build service; a tenant that sideloads gets its binary only from the tenant's deliverable script, which ends in the signature check (§3). |
| MUST | **"Own store" ≠ "own Firebase project."** A tenant's package can be an extra client inside the shared Firebase project (`google-services.json` carries many packages of one project). A separate file is needed only for a separate project, and a separate project is a backend workstream too (push credentials, ID-token acceptance, analytics); the docs state this so the decision is not taken by accident. |
| MUST | A **per-tenant release checklist** in the docs: config key, scripts, where the binary goes, who installs it, the signing-key registration status, the "all devices synced before a key change" step. |
| SHOULD | The runtime tenant list and the build-tenant list are compared in the audit; a build tenant with no runtime tenant (or the reverse) is listed with its reason (shared store build, legacy key). |
| SHOULD | Visual tenant differences (logo, card layouts) stay runtime-selected; the build axis changes identity only. Runtime assets kept in sync by array index across several `require` lists are a documented trap with a debug screen that shows every tenant's asset side by side. |

### 4. OTA updates (EAS Update / equivalent)

| Tier | Rule |
|---|---|
| MUST | If OTA is used: channels mirror environments (`preview`, `production`); the runtime version policy is `fingerprint` (preferred) or `appVersion`, so an OTA can never target an incompatible native binary. If OTA is **not** used, `release.md` says so and `eas.json` channels are documented as build-tagging only. |
| MUST | OTA publish happens from CI only: `preview` on merge to main, `production` as a separate manual-approval job (environment protection rule) after the store build is live. Profile B: a documented command run from the main branch after the store build is live, never from a feature branch. |
| MUST | A native change (new dep with native code, config plugin, SDK bump, a `patches/` change to a native file) changes the fingerprint/runtime version and therefore requires a store release; the pipeline fails the OTA job if the fingerprint changed and no build exists for it. |
| SHOULD | Rollback procedure documented and tested once (`eas update --republish` / channel rollback). |
| SHOULD | OTA messages carry the commit range (changelog section) for traceability. |

### 5. Release pipeline

| Tier | Rule |
|---|---|
| MUST | Driven by the release tool from `rn-project-standards.md` §7 (release-please PR merge, semantic-release on main, a pushed `vX.Y.Z` tag from commit-and-tag-version, or — profile B — the local `npm run release` on main). The pipeline **reads** the version; it never computes or bumps one. A legacy pipeline that derives the version from words in the commit message ("major"/"minor") is a finding. |
| MUST | Steps, in order, each gated on the previous: production build (per tenant) → signature verification → smoke E2E on the artifact (Level 3) → upload source maps + create Sentry release → submit to TestFlight / Play internal → (manual approval) promote to production track / phased release → production OTA channel switch (if applicable) → tenant deliverables handed over. |
| MUST | Store submission automated (`eas submit` / Fastlane `upload_to_testflight`, `upload_to_play_store`) with service-account / App Store Connect API key from secrets; the `submit` section of `eas.json` references key file **paths that are git-ignored**, with the committed placeholder clearly marked. |
| MUST | A GitHub Release / Azure DevOps release note (or equivalent) with the CHANGELOG section, build numbers, and links to the artifacts is created per version. |
| SHOULD | Phased/staged rollout (Play staged rollout %, App Store phased release) default on; a documented "halt rollout" step. |
| SHOULD | Hotfix path documented: branch from tag, `fix:` commit, patch release via the same pipeline; OTA-only hotfix when no native change. |

### 6. E2E in CI (testing Level 3)

| Tier | Rule |
|---|---|
| MUST | Maestro smoke tag runs on every preview build (main) and every release candidate, on the **built artifact** (Maestro Cloud / EAS Workflows Maestro step / self-hosted emulator+simulator job). |
| MUST | E2E failure blocks release promotion; it does not block PR merge (too slow) unless the `build-needed` path triggered a build. |
| MUST | Test accounts/seed data provisioned by the pipeline (script or API call) before the run; credentials from secrets. |
| SHOULD | Artifacts on failure (video, screenshots, Maestro logs) attached to the run; a chat notification with a direct link. |
| SHOULD | Full E2E set nightly on main. |

### 7. Quality gates & reporting

| Tier | Rule |
|---|---|
| MUST | Coverage uploaded (Codecov / Coveralls / CI summary) with a PR comment when a runner exists; the threshold in `jest.config` is the gate, the service is for visibility. At testing Level 0 the gate is the static checks and the repo-invariant scripts, surfaced as PR annotations. |
| MUST | `npm audit --audit-level=high` (or Snyk/Socket) in `validate` with a documented allowlist (`audit-ci` config) for known-unfixable transitive issues — not a permanently failing or permanently skipped job. |
| MUST | Lint annotations surface in the PR (GitHub problem matchers / reviewdog; Azure `##vso[task.logissue]` from an ESLint formatter), so reviewers see violations inline. |
| SHOULD | Bundle/app size tracking: JS bundle size per build (e.g. `expo export` size or `react-native-bundle-visualizer` summary) and APK/IPA size, posted as a PR comment or trend, with a soft budget. |
| SHOULD | Dependabot or Renovate configured for npm + CI actions, grouped minor/patch updates weekly, major updates as separate PRs; **Expo-managed packages excluded** (`rn-project-standards.md` §2 dependency policy — they move only with the SDK). |
| SHOULD | A nightly job runs `expo-doctor`/`rn doctor`, `npm outdated`, and `knip`/`depcheck`, opening an issue when something drifts. |

### 8. Workflow hygiene

| Tier | Rule |
|---|---|
| MUST | Workflows are small and composed: reusable workflows / composite actions (Azure: templates) for `setup` (checkout, node, cache, npm ci) used by every job; no copy-pasted setup blocks. |
| MUST | Every workflow has a one-line comment stating its trigger and purpose; job and step names are human-readable; a step that is kept but disabled says why in a comment. |
| MUST | `release.md` (from `rn-project-standards.md` §10) describes the profile, the pipelines or local scripts, how to trigger a preview build, how to promote, how to roll back, the tenant deliverables, the signing-key inventory and who approves. |
| SHOULD | Workflow files linted (`actionlint` for GitHub Actions; `az pipelines validate` or the YAML schema for Azure) in `validate`. |
| SHOULD | Status badges in README for validate and release. |

### Reference — GitHub Actions `validate` skeleton

```yaml
# .github/workflows/validate.yml — runs on every PR; must stay under 10 minutes
name: validate
on: { pull_request: {}, push: { branches: [main] } }
concurrency: { group: validate-${{ github.ref }}, cancel-in-progress: true }
jobs:
  validate:
    runs-on: ubuntu-latest
    timeout-minutes: 15
    steps:
      - uses: actions/checkout@v4
        with: { fetch-depth: 0 } # commitlint needs history
      - uses: actions/setup-node@v4
        with: { node-version-file: .nvmrc, cache: npm }
      - run: npm ci
      - run: npx commitlint --from ${{ github.event.pull_request.base.sha }} --to HEAD
        if: github.event_name == 'pull_request'
      - run: npm run typecheck
      - run: npm run lint
      - run: npm run format:check
      - run: npm run check-module-map && npm run check-doc-links && npm run compare-locales # the project's invariant scripts
      - run: npm run test:ci # only when a test runner exists
      - run: npx expo-doctor # or npx react-native doctor
      - run: npx audit-ci --config audit-ci.jsonc
      - uses: codecov/codecov-action@v4
        if: ${{ !cancelled() }}
```

### Reference — Azure Pipelines `validate` equivalent

```yaml
# azure-pipelines.validate.yml — PR validation; branch policy makes it required on main
trigger: none
pr: [main]
pool: { vmImage: ubuntu-latest }
steps:
  - checkout: self
    fetchDepth: 0
  - bash: echo "##vso[task.setvariable variable=NODE_VERSION]$(cat .nvmrc)"
    displayName: Read Node version from .nvmrc
  - task: UseNode@1
    inputs: { version: $(NODE_VERSION) }
  - task: Cache@2
    inputs: { key: 'npm | "$(Agent.OS)" | package-lock.json', path: $(npm_config_cache) }
  - bash: npm ci
  - bash: npm run validate
    displayName: typecheck, lint, format, invariants (, tests)
  - bash: npx expo-doctor
```

Secrets stay in a variable group marked secret and are referenced by name in `env:`; no step writes them into a committed file.

---

## Part 2 — Audit prompt (Claude Code)

```text
<<<<< PROMPT START >>>>>

Audit this React Native repository's CI/CD and delivery setup against "the rules" in `rn-cicd-standards.md` (Part 1). Read it first; stop and ask if missing. Also read `rn-project-standards.md` §1, §6, §7, §9, §10 if present, since this audit depends on them (pinning, hooks, commits, env, build profiles).

Do not modify workflows, build config or source until I approve in Phase 5. Docs and the migration plan may be written in Phase 4.

## Phase 0 — Inventory
Report with paths:
- Delivery profile evidence: CI platform(s) detected (`.github/workflows`, `azure-pipelines*.yml`, `.gitlab-ci.yml`, `bitrise.yml`, `.circleci`, `.eas/workflows`). For every pipeline file: trigger, jobs, steps, which steps are disabled, Node version, `npm ci` vs `npm install`, last modified date, whether anything in it is still the real release path. If you can read recent runs (`gh run list` / `az pipelines runs list`), report when it last ran. If nothing runnable exists, say "profile B".
- Local release path: release tool and script, build scripts per deliverable (`*:build:*`), config-stamping / tenant-apply script, signature verification script and whether the build scripts end in it, scratch-folder output, `eas.json` `cli.version`, `expo-doctor` state.
- Build system: `eas.json` profiles (and `appVersionSource`, `autoIncrement`, `channel`, env per profile, `extends`), or Fastlane (`fastlane/Fastfile` lanes, `Appfile`, `Matchfile`). How `expo.version`, `ios.buildNumber`, `android.versionCode` are set (release tool, EAS remote, stamping script, placeholders, hand-edited).
- Signing: generated `android/app/build.gradle` `signingConfigs` (does `release` use the debug keystore?), keystore inventory in the docs, which keys are registered with Firebase / attestation, whether two paths sign the same app id differently.
- Multi-tenant builds: tenant config module, apply script, per-tenant build scripts, Firebase file strategy, runtime tenant list vs build tenant list, per-tenant checklist.
- OTA: EAS Update config (`runtimeVersion` policy, channels) or "no OTA"; where publishes happen.
- Release tooling present and how it connects to CI (does a tag/PR trigger a build?) or to the local scripts.
- Node/Ruby/JDK pinning inside CI vs repo pin files; caching.
- Secrets handling: grep workflows for `echo`/`cat`/`printf`/`Set-Content`/`sed` of secrets, base64 blobs, placeholder replacement into committed files, secrets in `env:` passed to untrusted steps, fork-PR exposure; `eas.json` `submit` key paths and their gitignore status.
- Branch protection / policies (via `gh api` / `az repos policy list` if available, else ask).
- E2E in CI: Maestro/Detox jobs, what they run on, where artifacts go (or the testing level that defers it).
- Quality gates: coverage upload, audit, lint annotations, size tracking, Dependabot/Renovate config and whether Expo-managed packages are excluded.
- Docs: `release.md` or equivalent (profile stated? tenant checklists? key inventory?); README badges; existing agent-instruction files.
- Run `npm run validate` locally (if it exists, else the individual check scripts) and report pass/fail and duration.

## Phase 1 — Confirm detect-rules
State and ask me to confirm: the delivery profile (A / B) and, for B, whether moving to A is in scope; CI platform to target; EAS vs Fastlane; build-number mechanism (remote / CI / stamping script); OTA strategy (adopt EAS Update with fingerprint policy if absent? keep existing? none); credentials approach (EAS-managed / match / current); multi-tenant builds in scope and the tenant list; who approves production promotion; whether fork PRs are a concern (public repo?); what to do with a stale pipeline file (delete / header comment / revive). Wait.

## Phase 2 — Audit
Each rule: status ✅/⚠️/❌/➖, evidence, tier, size S/M/L/XL, files, risk (anything touching production submission, signing or OTA is high). Profile B: audit §0, §2 (local equivalents), §3, §3b, §4, §5 and §8's `release.md` row; list §1, §6, §7 as deferred with the trigger.
Sizing: S = a step or setting in an existing workflow, a script flag, a docs section; M = a new workflow or reusable setup action, a signature-check or stamping script, a tenant config module; L = build/OTA pipeline from partial to complete, E2E-on-artifact job, signing-key alignment with a user-base migration; XL = full pipeline from scratch including credentials and store submission, or migrating CI platforms (only noted, never recommended by this audit).

## Phase 3 — Gap report (chat)
1. Table MUST first; 2. totals; 3. top 5 by value/effort — typically: stale pipeline neutralised, signature check on every deliverable, build-number mechanism, `release.md` with the profile + key inventory + tenant checklists, validate workflow using the shared scripts + required check (profile A); 4. accepted exceptions; 5. **security findings first** (secrets written into committed files, debug-signed deliverable paths, unprotected main, fork PR access to secrets, debug keystore registered anywhere).

## Phase 4 — Docs + plan (write now)
- Agent-instructions file: "Delivery" section — the profile, which script CI (or the developer) runs, how a deliverable build is produced (the exact script per tenant), that production builds/OTA only come from the release path, never a debug-signed build to a user, never add secrets to workflows or committed files, path-filter rule for native builds (profile A).
- `release.md` (create or extend, in the docs location): profile, pipelines diagram (text) or local steps, triggers, promotion and approval, rollback (store + OTA), hotfix path, credentials and signing-key inventory, tenant build configs and checklists, who to contact. Stale pipeline files get their header comment.
- `cicd-migration-plan.md`: gap table with checkboxes; order: security fixes → stale pipeline neutralised → signature check + key inventory → build-number mechanism → `release.md` → (profile A) validate pipeline + branch protection → setup reuse/caching → preview build on main → OTA policy → release pipeline + submission → E2E on artifact → quality gates → nightly jobs; accepted exceptions.
Show a diff summary.

## Phase 5 — Ask before applying
Offer "everything" / "all MUST" / "pick by §" with sizes/risk and recommend. Default:
- Now: security fixes, stale-pipeline header or deletion, signature verification wired into every deliverable script, `release.md`, (profile A) `validate` workflow + required check, setup reuse, caching, concurrency, timeouts — low risk, no effect on builds.
- Separate PR, tested with a real preview or local build: build profiles, build-number mechanism, tenant apply script changes, fingerprint-based native-build trigger.
- Separate PR with a dry run before it is enabled on tags: release pipeline and store submission; OTA production job behind manual approval.
- E2E-on-artifact and nightly jobs last.
State clearly which items you cannot verify without a real build, store credentials or access to the signing service. Wait; re-ask if ambiguous.

## Phase 6 — Apply what I approved
- One workflow or one §item per commit; `ci:` / `build:` conventional commits in the project's convention.
- Validate every edited workflow with `actionlint` (install if missing) or the platform's validator before committing; for EAS, `eas config --profile <p>` to validate profiles where possible; for a stamping script, run it with `--skip-versioning` against a scratch copy and diff.
- Never commit secrets, never print them in logs; reference by name only. Never touch keystores or credential files.
- For anything that submits to stores, publishes OTA or signs a deliverable: add it disabled (`if: false` / manual `workflow_dispatch` / a script that refuses without an explicit flag) first, tell me how to do a dry run, and enable only after I confirm.
- Tick the plan; finish with a summary, what remains, and the exact manual steps I still need to do (credentials, environment reviewers, branch protection toggles, Firebase fingerprint registrations, tenant hand-over).

<<<<< PROMPT END >>>>>
```
