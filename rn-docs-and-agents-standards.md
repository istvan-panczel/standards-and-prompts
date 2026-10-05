# Project Documentation & Agent-Instructions Standards + Audit Prompt

> Standards version: 2 (2026-10-05). New in version 2; see `STANDARDS-INDEX.md` → Version history.

Part 1: rules (MUST / SHOULD). Part 2: audit prompt for Claude Code. Companion to the other `*-standards.md` files; **run it before any other file's Phase 4**, because every other audit writes documentation and must write it where this standard says.

Scope: how a repository documents itself for developers **and** coding agents — the entry file, the rule files, the maps and registers that keep docs equal to code, and the working rules an agent follows in the repo. The standard treats the two audiences as one: a rule file that an agent can follow is a rule file a new developer can follow.

---

## Part 1 — Rules

### 1. Shape of the documentation system

```
README.md                       # local development only: requirements, install, run, before-you-commit, links
CLAUDE.md (or AGENTS.md)        # THE entry file: critical rules, critical don'ts, commands, decision tree, routing
AGENTS.md / GEMINI.md / …       # pointers to the entry file for other agents; never hold rules of their own
NPM_SCRIPTS.md                  # every script: what, when, prerequisites (rn-project-standards.md §2)
<docs>/                         # detect: docs/, claude/, doc/ — one tree, whatever its name
  rules/<area>/<topic>.md       # rule files: one topic each, "Read this file when:" header
  documentation/module-map.md   # every src/ folder has an entry (§2)
  documentation/navigation-atlas.md
  documentation/<integration>.md
  open-follow-ups.md            # known open items (§4)
  upgrade-followups.md          # dependency policy + open upgrade items (§4)
  plan/<initiative>/README.md   # living plan of a long migration, only when one is running (§5)
  tasks/                        # task description files for the task workflow (§6), gitignored or kept
.claude/commands/*.md           # slash commands for the task workflow (§6)
temp-local/                     # gitignored scratch (§7)
```

| Tier | Rule |
|---|---|
| MUST | **One entry file** (`CLAUDE.md`, or `AGENTS.md` where the team prefers) holds the rules that apply everywhere and the routing to the detailed docs. It is loaded into every agent session, so it is short: one or two lines per rule, detail in a rule file. Other agents' files (`AGENTS.md`, `GEMINI.md`, `.cursorrules`, `copilot-instructions.md`) are **pointers** to it and say so; a rule written in a pointer file is a gap. Tool-specific sections (which LSP, which MCP servers, which terminal) are marked as such so other tools map them to their equivalents. |
| MUST | **One docs tree**, detected and kept (`docs/`, `claude/`, …). Rule files are grouped by area (patterns, technical, ui, naming, workflows) and each opens with **"Read this file when:"** so the reader can stop after one line. A documentation file for an integration (a partner app, a signing service, a confirmation server) lives in `documentation/`, not in `rules/`. |
| MUST | **The README is the local development guide and nothing else**: requirements with versions and why, install, first run, daily start, when the dev client must be rebuilt, what the app asks for on first launch, what to run before committing, and links to the entry file, the module map and the scripts guide. Everything about the code lives in the docs tree. A README that explains architecture is a README that is wrong within a quarter. |
| MUST | **The entry file has a decision tree**: "finding the code for a feature → module map; adding a table → …; new screen → …; fixing a bug → testing doc + open follow-ups; upgrading → upgrade follow-ups; code review → code quality". Routing by task, not by file name. |
| MUST | **Critical rules are numbered and never renumbered.** The entry file lists the rules that are referenced from other docs and from code comments (`CLAUDE.md rule #15`) by number; a retired rule keeps its number with "retired, see …". **Critical Don'ts** are a separate list of things that look reasonable and are wrong here (the backend-driven-data guard, the navigation-params merge, the dependency bump past the SDK, the debug-signed build). |
| MUST | **A recurring trap gets a rule.** When a bug class can come back silently, it becomes a numbered rule or a don't in the entry file with its detail in a rule file, **and** a lint rule or a repo-invariant script where one is possible (`rn-project-standards.md` §4, §6). A doc alone is the weakest form; the entry-file line points at the enforcement. |
| SHOULD | Each rule file ends with a **"Related files"** list and, where the topic is a recipe, a **checklist**; reference implementations are named by path and symbol ("canonical example: `src/x/store/x-saga.ts`, `syncX`"). |
| SHOULD | A **"Which rule file to read"** table in the entry file mirrors the decision tree in a scan-friendly form (file → read when). |

### 2. Maps and registers that keep docs equal to code

| Tier | Rule |
|---|---|
| MUST | **Module map**: every folder directly under `src/` has one entry (purpose, the business names users and tickets use in their own language, status such as active / infrastructure / debug only / legacy, which tenants or gates use it, screens and their stack, slices and whether persisted, tables, endpoints with uploads marked, entry-point files, the doc that covers it, traps). Large apps split it into an index plus area files. **A pre-commit script fails when a `src/` folder has no entry, an entry has no folder, an entry appears twice, or an index row is missing.** A new, renamed or removed folder updates the map in the same change. |
| MUST | **Doc link check**: a pre-commit script verifies that relative Markdown links point to existing files or folders, that `#anchors` match a heading of the target, that **source paths written as inline code** (`src/feature/file.ts`) exist, and that code or tooling files which mention a doc path point at an existing doc. Text in fenced code blocks and paths with placeholders are skipped; plan files may describe files that do not exist yet and are exempt from the source-path check; the scratch folder is skipped. |
| MUST | **Docs reference code by path and symbol, never by line number** (`shopListSaga.ts`, `createShopList`), because line numbers rot on the next edit while the link checker cannot see them. |
| MUST | **Only verified facts.** A doc states what the author saw in the code, on the device or in a response. Backend behaviour the code cannot prove is **worded as reported** ("the backend is reported to…", with the date and ticket). Dated "verified on" lines on anything that drifts (toolchain guides, current-state sections). |
| MUST | **No personal or machine data in docs**: no names, home paths, device ids, IP addresses, DSNs, credentials, customer-internal URLs. Roles instead of names ("the project owner", "the backend team"); placeholders instead of hosts. |
| MUST | **Docs are formatted and checked like code**: Prettier covers the Markdown (`rn-project-standards.md` §5), the link and map checks run in pre-commit, and a change that moves, renames or deletes a file updates every doc that names it in the same change. |
| SHOULD | A **navigation atlas**: every registered route, its stack or tab, who navigates to it and with what params, kept current when a flow changes. A **hooks catalogue** and a **component catalogue** with props and per-platform traps. A **scripts guide** (`rn-project-standards.md` §2). |
| SHOULD | **Generated reference files** (DDL dumps from a real device DB, API type outputs) are documentation, regenerated by a script, never hand-edited; a stale one is listed as an open follow-up, not fixed by hand. |
| SHOULD | A **"Names that do not match"** table in the module map: the business name → the folder, when they differ (the promotions tab that is the discounts module; the Impl suffix that is legacy); the mismatches are the most common way to open the wrong folder. |

### 3. The project docs are the record

| Tier | Rule |
|---|---|
| MUST | **Durable findings go into the project docs**, where every developer and every agent can read them: backend contracts, bug classes, gotchas, design decisions, accepted exceptions to these standards. A coding tool's private memory or a chat transcript may help one agent recall, but **must never be the only place a finding is written down, and project docs never point to such notes.** |
| MUST | **Docs are updated in the same change** as the code they describe: a new upload gets its checklist entry; a changed flow updates the atlas; a fixed bug removes its follow-up item. "Update the docs" is a step of every task, named in the plan (§6). |
| MUST | The docs say what **is**, not what was: history is kept only where it explains the current state (a dated "History" section at the bottom of an upgrade doc; a "why" paragraph next to a decision). Superseded instructions are deleted, not struck through. |
| MUST | **Accepted exceptions are documented next to the rule they bend**, with the reason and the scope ("`Pressable`-based controls in high-frequency lists only; everywhere else rule #5 applies"). An exception without a scope becomes the new default within a quarter. |
| SHOULD | Every rule file lists **the lint rules and scripts that enforce it**, so the reader knows what review has to catch and what the tooling already does ("not enforced by lint, review enforces: …"). |
| SHOULD | A rule that bit twice gets a one-line **anti-patterns table** entry (area → anti-pattern → fix) in its rule file; the table is the quickest scan for a reviewer. |

### 4. Open work has a home

| Tier | Rule |
|---|---|
| MUST | **`open-follow-ups.md`** lists every known open item that no other tracker holds: unfixed bugs, latent gaps and deferred clean-ups, items waiting on the backend (worded as reported), decisions pending, platform/tooling issues, items relevant once a branch merges. Each item: what, where (path + symbol), evidence, the ticket once one exists, a verified-on date. An index at the top groups them. **Remove an item in the same change that fixes it**; add one when a review or an investigation finds something out of scope. |
| MUST | **`upgrade-followups.md`** (or a section) holds the **dependency policy** (`rn-project-standards.md` §2), a dated **current state** (SDK, RN, React, TypeScript versions; architecture flags; compiler on/off; patches carried with their reason and exit plan), the open upgrade items, the known harmless warnings with their cause ("do not re-investigate"), the **no-op / blocked list** ("don't re-ask"), and the upgrade checklist. Items that are not upgrade-related link to `open-follow-ups.md` rather than duplicating. |
| MUST | **Debugging a symptom starts in `open-follow-ups.md`**: the entry file's decision tree routes "fixing a bug" there and to the testing doc, because the item may be the bug being looked at, or the change may fix it. |
| SHOULD | A **"Waiting on backend"** item records what the client sends, what the sync should deliver, and how to verify it from the device (live state path, DB query), so the next person can re-check in minutes. |
| SHOULD | A **"Decisions pending"** item states the options and who decides; an agent that meets the decision asks instead of choosing. |

### 5. Plans and initiatives

| Tier | Rule |
|---|---|
| MUST | **Plan first, in the conversation.** For a non-trivial change the agent presents the plan (summary, files with one line of reason each, order, who does what, risks and out-of-scope, verification, docs to update) and waits for approval before writing code. **No plan file is written into the repo unless the owner asks for one**; plans agreed in conversation are not persisted. |
| MUST | A **long-running initiative** (a UI-library migration, a data-layer rewrite) that spans many sessions gets **one living plan folder** (`plan/<initiative>/README.md` + supporting docs): why, the end state, **locked decisions** with their reasons, phases with status, risks and watch-outs, open questions with owners, gotchas learned, linked documents, and a "how to use this doc" section. It is updated throughout the initiative and is the single source of truth for it; the entry file points at it from a banner while it runs. |
| MUST | Rules that depend on an initiative's state are **tagged** in the rule files (`[NB]` / `[RNR]` / `[both]` during a UI migration) and a "which to use for new code" table decides per situation (`rn-architecture-standards.md` §4). |
| SHOULD | A plan folder has a `TODO.md` for the initiative's small items and a `components/<name>.md` (or equivalent) per migrated unit with its decisions, so the entry README stays readable. |
| SHOULD | When an initiative ends, its durable knowledge moves into the rule files and the plan folder is deleted or reduced to a history note. |

### 6. How an agent works in the repo

These are the working rules the entry file states for coding agents. They are the rules a careful colleague follows too.

| Tier | Rule |
|---|---|
| MUST | **Read the rule file for the task before writing code.** The entry file routes by task (§1); "the rule file" is the detail, the entry file only the pointer. Reviews re-read the rule files instead of working from memory, because they change. |
| MUST | **Ask before architectural forks.** When two defensible approaches exist (tactical vs structural, lazy vs eager, workaround vs refactor, two existing patterns that could apply), the agent presents both with trade-offs and a recommendation and lets the owner decide. Clear-cut choices need no question; questions the code or the docs can answer are looked up, not asked. |
| MUST | **Debug tooling is asked about** (`rn-architecture-standards.md` §12): when a task does not specify mock data, a debug screen, saga mocking or a flag override, the agent asks. |
| MUST | **Git is the owner's.** No commits, branches, pushes, stashes or resets unless explicitly asked. When asked to commit: the project's header convention (`rn-project-standards.md` §7), and the team's **tool-attribution decision** followed exactly (whether `Co-Authored-By` / "generated with" trailers are allowed). |
| MUST | **Never call the backend directly while investigating** (`curl` with the app's token and the like) unless the owner asks; inspect what the app received (live state, persisted store, extracted DB, logs). Backend traffic comes only from the app. |
| MUST | **No client-side guards against backend-driven data** in the profiles where the backend owns the truth (`redux-saga-best-practices.md` §9 profile C): the reflex to add a duplicate check is a rule violation here, and the entry file says so in its don'ts. |
| MUST | **Verification is reported, not claimed** (`rn-testing-standards.md` §0): real output of the static checks, what was checked on which platform, what was not verified, what the owner must check by hand. At Level 0 the agent drives the simulator per the guide where it can. |
| MUST | **Orchestration without abdication**: for work of any size the agent plans, delegates exploration and larger implementation to sub-agents where that helps, **reviews every sub-agent's output before accepting it**, and makes small well-understood edits directly rather than spawning an agent for them. Model choice per task type is stated in the entry file (exploration / complex implementation vs well-specified low-complexity work). |
| MUST | **Task workflow as slash commands** (`.claude/commands/` or the tool's equivalent) when the team works from task files: `new-task <file>` (understand the task, read the routed rule files and the module-map entries, read `open-follow-ups.md`, ask, plan in conversation, implement after approval, verify, update docs, leave git alone), `new-task-review <file> <staged \| commit <sha> \| branch <base>>` (read-only review: scope statement, requirements mapped to code with path + symbol, plan deviations, **regressions via references of every changed symbol**, project rules re-read from the rule files, backend-driven design, docs to update), `task-reformat <file>` (clean a task description so a later session can plan from it). The commands point at the entry file and add only what is task-specific. |
| SHOULD | Ticket numbers are the **search key**: in commit headers (option B convention), in code comments that exist because of a ticket (`// TODO: … (#4885)`), in Sentry messages, in follow-up items and in docs. Business feature names are in the glossary (`rn-runtime-quality-standards.md` §D) and the module map. |
| SHOULD | **testID in touched code** is part of every change (`rn-architecture-standards.md` §2) and listed in the agent's review checklist. |
| SHOULD | The entry file names the tools the agent should prefer (TypeScript LSP for symbols over text search; grep for strings; a docs MCP for library versions; the device-driving guide) and the files that must not be edited by hand (generated DDL, patches without the regen script, the lockfile). |

### 7. Scratch space and dev tooling

| Tier | Rule |
|---|---|
| MUST | A gitignored **scratch folder** (`temp-local/`) for extracted databases, screenshots, exported state, local build outputs and personal notes. Scripts write there by default; the doc checks skip it; nothing project-relevant lives only there (§3). Standards files being drafted may live there until they are adopted. |
| MUST | **Dev tooling scripts are documented where they are used**: the testing doc for state and DB extraction, the simulator guide for device driving, the scripts guide for every npm entry. Each script has a header comment with what it does, how to run it, exit codes, and whether a hook runs it. |
| SHOULD | A **simulator / emulator interaction guide** with a verified-on date and toolchain versions (`rn-testing-standards.md` §0); its known quirks are dated and re-verified after a toolchain upgrade. |
| SHOULD | **MCP servers and agent tools under evaluation** are listed with the blocker that defers them ("after the debugger crash fix in RN x.y"), so the question is not re-opened every session. |

---

## Part 2 — Audit prompt (Claude Code)

```text
<<<<< PROMPT START >>>>>

Audit this repository's documentation system and agent-instructions against "the rules" in `rn-docs-and-agents-standards.md` (Part 1). Read it first; stop and ask if missing.

Do not modify anything except documentation, the migration plan file and documentation scripts until I approve in Phase 5. This audit writes no application code.

## Phase 0 — Inventory
Report with paths:
- Entry file(s): CLAUDE.md, AGENTS.md, GEMINI.md, .cursorrules, copilot-instructions — which one holds rules, which are pointers, and whether any pointer holds rules of its own. Length of the entry file; does it have numbered critical rules, critical don'ts, commands, a decision tree, a rule-file table, tool-specific sections marked as such?
- Docs tree: location and structure; rule files with and without a "Read this file when" header; documentation vs rules separation; checklists and related-files sections; anti-pattern tables.
- README contents: local-development only, or architecture/rules mixed in; requirements with versions; before-you-commit section; links to the docs.
- Module map: present? coverage (run the check script if present, else diff `src/` folders against the map); entry fields present; "names that do not match" table; area split.
- Doc link check script: present? what it checks (links, anchors, inline source paths, code→doc references); run it and report.
- Navigation atlas, hooks catalogue, component catalogue, scripts guide: present and current (spot-check three routes, three hooks, three scripts).
- Line-number references in docs (grep `:\d+` after a path, "line \d+"); personal or machine data (names, home paths, IPs, DSNs, device ids); "reported" wording on backend claims; verified-on dates on toolchain guides.
- Open work: `open-follow-ups.md` (structure, index, dates, tickets, items whose referenced code no longer exists), `upgrade-followups.md` (policy, dated current state, patches table, harmless warnings, no-op list, checklist).
- Plans: plan folders present; are they living (dated updates, locked decisions, phases with status) or stale; does the entry file ban unrequested plan files; is a running initiative bannered in the entry file and tagged in the rule files?
- Agent working rules in the entry file: plan-first, ask-before-forks, debug-tooling question, git policy and attribution decision, no direct backend calls, backend-driven-data don't, verification reporting, orchestration guidance, tools to prefer, files never hand-edited.
- Task workflow: slash-command files present (`new-task`, review, reformat or equivalents), their modes, whether they re-read rule files and check regressions via references.
- Scratch folder: present, gitignored, skipped by the doc checks; scripts default to it.
- Formatting: does `format:check` cover the docs; do hooks format staged Markdown.

## Phase 1 — Confirm detect-rules
State and ask me to confirm: which file is the entry file and which are pointers; the docs tree location to keep; whether to introduce a module map (and its split) and the two check scripts; the attribution decision for commits; whether the task-file workflow is wanted; whether a running initiative should get a plan folder (only if one is running and none exists); what to do with stale plan files. Wait.

## Phase 2 — Audit
Each rule: status ✅/⚠️/❌/➖, evidence, tier, size S/M/L/XL, files, risk (low for docs; medium when a script is wired into pre-commit and may block commits until the map is complete).
Sizing: S = a section or a header line; M = a script (module map check, doc link check) with the first full pass to make it green, or the open-follow-ups file seeded from a code sweep; L = a module map for a large app (every `src/` folder researched and described), a navigation atlas, splitting a bloated README into rule files; XL = a documentation system from nothing on a large legacy app (do it area by area, map first).

## Phase 3 — Gap report (chat)
1. Table MUST first; 2. totals; 3. top 5 by value/effort — typically: entry file with numbered rules + decision tree, module map + check script, doc link check, open-follow-ups seeded, agent working rules (git, backend calls, verification report); 4. accepted exceptions; 5. anything that is **wrong right now** (a doc naming a deleted file, a line-number reference, personal data, a pointer file with its own rules) at the top.

## Phase 4 — Docs + plan (write now)
- Entry file: create or restructure to the shape in §1 (keep every existing rule; move detail to rule files; number the critical rules; add the don'ts, the commands, the decision tree, the rule-file table, the agent working rules of §6 in one short section). Convert other agents' files into pointers.
- Docs tree: add "Read this file when" headers, related-files lists; move architecture text out of the README; create `module-map.md` (+ area files) with every `src/` folder — research each one; create `open-follow-ups.md` and `upgrade-followups.md` seeded from TODO comments, disabled tests, stale pipeline files, patches, known warnings; create the scripts guide if scripts > 15.
- Scripts: `check-module-map` and `check-doc-links` (Node, header comment, exit codes) wired into pre-commit; `format:check` extended to the docs. Run both until green.
- `docs-migration-plan.md`: gap table with checkboxes; order: entry file → pointers → link check (fixes what is wrong now) → module map + check → follow-up files → README trim → atlas/catalogues → task workflow commands → plan-folder hygiene; accepted exceptions.
Show a diff summary.

## Phase 5 — Ask before applying
Offer "everything" / "all MUST" / "pick by §" with sizes and recommend. Default:
- Now: entry file restructure, pointer files, doc link check + the fixes it finds, agent working rules, formatting of docs, scratch folder.
- Next, as one PR: module map + check script (block commits only once the map is complete), follow-up files.
- Then: README trim, atlas and catalogues, task commands.
Wait; re-ask if ambiguous.

## Phase 6 — Apply what I approved
- One §item per commit in the project's commit convention (when I ask you to commit; otherwise leave git to me).
- Never delete a rule while restructuring; move it and leave a pointer if another doc or a code comment references its old place (the link check tells you).
- Only write what you verified in the code or on the device; word backend behaviour as reported; no personal or machine data.
- Run `check-doc-links`, `check-module-map` and `format:check` after each item and show the output.
- Tick the plan; finish with a summary, what remains, and the manual decisions still open (attribution policy, stale plan files, initiative banners).

<<<<< PROMPT END >>>>>
```
