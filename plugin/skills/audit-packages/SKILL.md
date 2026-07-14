---
name: audit-packages
description: "Audits a project's dependencies (PHP composer and npm/yarn/pnpm JS): lists available updates classified by impact level (minor with no risk, medium requiring some adjustments, critical with breaking changes), detects known security vulnerabilities (CVE), and generates a tracking report. Use this skill as soon as the user requests a dependency audit, a check of package versions, a search for security vulnerabilities in dependencies, an overview of available updates, or mentions words like 'outdated', 'vulnerabilities', 'CVE', 'package update', 'dependency audit', 'npm audit', 'composer audit', even without an explicit request."
argument-hint: "[--scope=php|js|all] [--auto-save] [--quiet]"
allowed-tools: Read, Write, Edit, Glob, Grep, Bash(composer *), Bash(npm *), Bash(yarn *), Bash(pnpm *), Bash(dock *), Bash(docker *), Bash(git *), Bash(jq *), Bash(cat *), Bash(ls *), Agent(explore-codebase), WebFetch, Skill
accept-on: Edit
---

# Audit Packages

## Output language

Produce all user-facing content (messages, generated documents, titles, descriptions, commit messages, changelog, summaries) in the user's language: match the language the user writes in (French or English). If undetermined, default to English. These instructions are written in English and do not dictate the output language.

Audit the project's dependencies and classify updates by impact level: **$ARGUMENTS**

## Objective

Provide a clear view of the state of a project's dependencies, without asking for unnecessary confirmation:

1. **Identify** outdated packages (composer, npm/yarn/pnpm) — including vulnerable transitive ones
2. **Classify** updates by impact level (minor, medium, critical)
3. **Detect** known security vulnerabilities (CVE) in installed versions
4. **Display** the report directly in the console (no file by default)
5. **Propose** concrete actions at the end: markdown save, applying minor updates, workplan for critical ones

## Options

- `--scope=php|js|all`: Limit the audit to one ecosystem (default: `all`)
- `--auto-save`: Save the markdown report without asking (default: ask after display)
- `--quiet`: Suppress intermediate messages ("collecting…", etc.)

## Interaction philosophy

This skill favors **direct action** over intermediate confirmations:

- **Do not ask** before running `composer outdated`, `composer audit`, `npm outdated`, `npm audit`: these are reads, with no side effects.
- **Do not ask** before creating output *directories* (`docs/audits/`) — but **do ask** before writing a file there.
- **Display** the console report as soon as it is ready. **Then** ask a single overall question:
  *"What do you want to do? 1) save as markdown  2) apply the minor updates  3) create a workplan for a critical one  4) nothing"*.
- The `--auto-save` / `--scope` options let you pre-answer and short-circuit the question.

## Context

- Current branch: !`git branch --show-current`
- Modified files: !`git status --porcelain`

## Workflow

### 1. Environment detection

First of all, identify what is available in the project:

| File detected | Ecosystem | Manager |
|---|---|---|
| `composer.json` | PHP | composer |
| `package.json` + `pnpm-lock.yaml` | JS | pnpm |
| `package.json` + `yarn.lock` | JS | yarn |
| `package.json` + `package-lock.json` | JS | npm |
| `package.json` (no lock) | JS | npm (default) |

Consult `CLAUDE.md`, `AGENTS.md` and `README.md` to determine:
- The execution environment (Docker via `dock`, a specific wrapper, direct execution)
- Project constraints (PHP version, Node version)
- Update conventions (version freeze, CI requirements)

**Adapt** all the commands below accordingly (e.g. `dock composer outdated` rather than `composer outdated`).

### 1.1 Execution fallback strategy

The audit only reads the state of the packages. It must work even if the usual environment is unavailable:

1. **Project wrapper detected** (`dock`, `docker-compose exec`, alias) → try first.
2. **Wrapper failure** (e.g. stopped container: `No such container`) → **silently fall back** to local execution (`composer` / `npm` available in the PATH) and mention it at the end of the report as a discreet line: `Run locally (Docker container stopped)`.
3. **Composer / npm missing from the PATH** → inform the user and offer to start the containers (`docker-compose up`) or install locally.

**Never block the audit** to ask the user to start Docker if local execution is possible. The results are identical: the basis is `composer.lock` / `package-lock.json`, independent of the runtime.

### 2. Data collection

#### 2.1 PHP packages (composer)

If `composer.json` is present:

```bash
# List outdated packages (JSON for reliable parsing)
composer outdated --direct --format=json

# Security audit (available since Composer 2.4)
composer audit --format=json
```

Capture:
- **Outdated**: name, current, latest, latest-status
  - `up-to-date` → nothing to do
  - `semver-safe-update` → the latest version already satisfies the `composer.json` constraint (equivalent to a **minor** with no manifest change)
  - `update-possible` → a newer version exists but the constraint must be relaxed (**major jump** or restrictive constraint)
- **Audit**: advisories (CVE, severity, title, link, affected versions) + `abandoned` (abandoned packages)

#### 2.2 Surfacing vulnerable transitive dependencies

`composer audit` CVEs sometimes affect **transitive** packages (not listed in `composer.json`). To make them actionable:

```bash
# Identify what introduces the vulnerable package
composer depends <vendor/package>
```

For each vulnerable transitive, capture the **direct package(s)** that pull it in (the *Via* column in the report). The update is done at the direct package level, not the transitive one.

#### 2.3 JS packages (npm/yarn/pnpm)

If `package.json` is present, use the detected manager:

**npm**:
```bash
npm outdated --json
npm audit --json
```

**yarn v1**:
```bash
yarn outdated --json
yarn audit --json
```

**yarn berry (v2+)**:
```bash
yarn npm audit --json
yarn outdated --json  # via plugin or fallback
```

**pnpm**:
```bash
pnpm outdated --format json
pnpm audit --json
```

Capture: name, current, wanted (result after `npm update` respecting the constraint), latest (latest published), type (dependencies/devDependencies).

**Common case to flag**: `wanted == current`. This means the `package.json` constraint (e.g. `^2.1.8`) forbids any update, even a patch. The package then appears directly as 🔴 Critical (the constraint must be relaxed to fix the CVEs).

JS CVEs (`npm audit`) are often on **transitive** dependencies (chain `chokidar → anymatch → micromatch → braces`). The `fixAvailable` field indicates the **direct** package on which the update must be done, and whether it is `isSemVerMajor`.

### 3. Update classification

For **each** outdated package, determine its impact level by combining several signals:

#### MINOR level (0 impact, applicable immediately)

Conditions (all of them):
- **patch** or **minor** version jump per semver (`1.2.3` → `1.2.4` or `1.3.0`)
- Satisfies the current `composer.json` / `package.json` constraint (`^` or `~`)
- No vulnerability detected on the current version (or CVE fixed by this update)
- No mention of a breaking change in the changelog (to verify quickly if in doubt)

Action: **can be applied without changing the code**.

#### MEDIUM level (some adjustments)

Conditions (at least one):
- **minor** jump but the package is critical (framework, ORM, router, etc.)
- Presence of announced deprecations (`@deprecated`, runtime warnings)
- The changelog mentions changed behavior requiring verification
- A **low/medium** severity vulnerability that forces the update
- Package little used in the code but critical to functionality (e.g. payment lib)

Action: **read the changelog, run the update locally, run the test suite, adjust if needed**.

#### CRITICAL level (breaking changes, workplan required)

Conditions (at least one):
- **major** version jump (`1.x.x` → `2.0.0`)
- **high/critical** severity vulnerability
- Package at the heart of the architecture (Laravel, Symfony, React, Vue, Node) whose migration touches >10 files
- Changelog mentions documented breaking changes (API removal, changed signatures)
- Constraint to relax in the manifest file (e.g. moving from `^9.0` to `^10.0`)

Action: **create a dedicated migration workplan** via the `/workplan` skill.

#### Classification heuristic

For each package, produce a feasibility score `Easy | Moderate | Complex`:

| Criterion | Easy | Moderate | Complex |
|---|---|---|---|
| Semver jump type | patch / minor | notable minor | major |
| Number of files potentially impacted | < 5 | 5-20 | > 20 or unknown |
| Presence of breaking changes | none | a few, documented | many or poorly documented |
| Active vulnerability | none | low/medium | high/critical |
| Test coverage involved | good | partial | weak |

Use `Agent(explore-codebase)` to estimate the number of files using a given package when needed (e.g. `grep -r "use PackageName" src/`).

For ambiguous cases, consult the package changelog via `WebFetch` (Packagist, GitHub releases, npm registry) rather than guessing.

### 4. Report generation

The report is **always displayed in the console first**. The markdown file is only created after validation (or automatically with `--auto-save`).

#### 4.1 Console report

**Structure**:

```
Dependency audit — 2026-04-15
Project: my-laravel-project
Scope: PHP + JS

══════════════════════════════════════════════════════════════════════

📦 PHP (composer) — 12 outdated direct packages

Package                        Current      Latest        Status             Level        Feasibility
─────────────────────────────────────────────────────────────────────────────────────────────────────
laravel/framework              10.48.4      10.48.28      semver-safe        🟡 Medium    Moderate
phpunit/phpunit                10.5.2       10.5.45       semver-safe        🟢 Minor     Easy
pestphp/pest                   2.34.0       2.36.0        semver-safe        🟢 Minor     Easy
guzzlehttp/guzzle              7.8.0        8.0.0         update-possible    🔴 Critical  Complex
sentry/sentry-laravel          4.1.0        4.10.0        semver-safe        🟢 Minor     Easy
…

🔐 PHP vulnerabilities — 3 advisories (1 HIGH, 2 MEDIUM)

 Severity  Package                  Via (direct)              CVE / Issue
─────────────────────────────────────────────────────────────────────────
 HIGH      symfony/http-foundation  my-framework (transitive) CVE-2024-XXXXX
 MEDIUM    league/flysystem         direct                    CVE-2024-YYYYY
 MEDIUM    league/flysystem         direct                    (second advisory)

🪦 Abandoned: old-lib/foo (replaced by new-lib/foo)

══════════════════════════════════════════════════════════════════════

📦 JS (pnpm) — 8 outdated direct packages

Package                Current     Wanted      Latest        Level        Feasibility
─────────────────────────────────────────────────────────────────────────────────────
vue                    3.4.15      3.4.38      3.4.38        🟢 Minor     Easy
vite                   5.1.0       5.4.10      6.0.0         🟡 Medium    Moderate
typescript             5.3.3       5.6.3       5.6.3         🟢 Minor     Easy
vuex                   4.1.0       4.1.0       4.1.0         🔴 Critical  Complex (deprecated)
chokidar               2.1.8       2.1.8       5.0.0         🔴 Critical  Moderate (blocking constraint)
…

Note: "wanted == current" means the package.json constraint
blocks any update — relax the constraint to fix the CVEs.

🔐 JS vulnerabilities — 1 advisory (1 HIGH)

 Severity  Package     Via (direct)        CVE / Issue
─────────────────────────────────────────────────────────────────────────
 HIGH      axios       direct              CVE-2024-ZZZZZ

══════════════════════════════════════════════════════════════════════

Summary:
  🟢 Minor     : 15 packages (direct application possible)
  🟡 Medium    :  3 packages (checks required)
  🔴 Critical  :  2 packages (workplan recommended)
  🔐 Vulns     :  4 CVE (incl. 2 HIGH to address quickly)
  🪦 Abandoned :  1 package

Run locally (Docker container stopped).
```

**Formatting rules**:

- Composer columns: `Current | Latest | Status | Level | Feasibility`
  - `Status` faithfully reflects the output of `composer outdated` (`semver-safe`, `update-possible`, `up-to-date`). Do not invent a separate "possible minor version" — when `status = semver-safe`, the `Latest` column **is** the applicable minor version.
- npm columns: `Current | Wanted | Latest | Level | Feasibility`
  - `Wanted` = result of an `npm update` without modifying `package.json`. If `Wanted == Current`, flag it explicitly (note under the table).
- Vulnerabilities section: always display the **Via (direct)** column to distinguish direct CVEs from transitive ones.
- Icons 🟢 🟡 🔴 🔐 🪦 for visual spotting.
- Package names truncated to 32 characters + `…` if needed.
- Discreetly note at the end of the report if execution was done locally rather than via the Docker wrapper.

#### 4.2 Markdown file (on request only)

The file is only generated **after displaying the console report**, if the user confirms (or if `--auto-save` is passed). Path: `docs/audits/packages-YYYY-MM-DD.md` (create the folder if absent, overwrite a file from the same day after confirmation).

Enriched structure:

```markdown
# Dependency audit — YYYY-MM-DD

**Project**: <project-name>
**Branch**: <git-branch>
**Commit**: <short-sha>
**Scope**: PHP + JS

## Summary

| Level | Count | Recommended action |
|---|---|---|
| 🟢 Minor | 15 | Direct application (see § Minor) |
| 🟡 Medium | 3 | Verification + tests (see § Medium) |
| 🔴 Critical | 2 | Migration workplan (see § Critical) |
| 🔐 Vulns | 3 | HIGH priority to address immediately |

## PHP (composer)

### Minor updates — 🟢

| Package | Current | New | Constraint | Changelog |
|---|---|---|---|---|
| ... | ... | ... | ... | [link](...) |

### Medium updates — 🟡

For each package:
- **Name**: current version → target version
- **Reason for classification**: deprecations, changelog, tests to adjust
- **Potentially impacted files**: indicative list
- **Changelog**: link

### Critical updates — 🔴

For each package:
- **Name**: current version → target version
- **Breaking changes identified**: list
- **Impact estimate**: number of files, affected areas
- **Associated workplan**: `docs/workplans/upgrade-<package>.md` (if created)

### Vulnerabilities

| Severity | Package | CVE | Version | Fix | Link |
|---|---|---|---|---|---|
| HIGH | ... | CVE-... | ... | ... | [advisory](...) |

## JS (<manager>)

Same structure as the PHP section.

## Suggested next steps

1. Apply the 15 minor updates via `composer update` / `<manager> update`
2. Plan the 3 medium updates (dedicated branch, PR per package)
3. Create workplans for the 2 critical updates
4. Address the 2 HIGH CVEs as a priority

---

*Report generated on YYYY-MM-DD HH:MM by the `/audit-packages` skill.*
```

Make sure the `docs/audits/` folder exists, otherwise create it.

### 5. Post-report actions — a single question

After displaying the console report, ask **a single grouped question** with all the available actions (depending on what the report contains). Example:

```
What do you want to do?
  [s] Save the report as markdown (docs/audits/packages-YYYY-MM-DD.md)
  [m] Apply the 5 minor updates (composer update <list>)
  [w] Create a workplan for a critical package (specify which one)
  [v] Address the HIGH/CRITICAL CVEs as a priority (summary of possible actions)
  [q] Nothing, I've seen what I wanted
```

Rules:

- Only display the **relevant** choices (no `[m]` if there is no minor, no `[w]` if there is no critical).
- Accept combinations (`sm`, `sw`, etc.) — the user may want to save **and** apply the minor updates right away.
- If `--auto-save` is passed, pre-execute the `[s]` action without asking.
- If the user chooses `[w]`, just ask for the package name then trigger `Skill(skill="workplan")` with a pre-filled prompt like:
  > Migrate the package `X` from version `A.B.C` to `X.Y.Z`. Documented breaking changes: <changelog summary>. Identified dependent packages: <list>. Vulnerabilities to fix: <CVE list if present>.
- If the user chooses `[m]`, directly run `composer update --with-dependencies <list>` and/or `npm update <list>` without asking again. Then run the verification (§ 6).
- If the user chooses `[v]`, present a mini-plan of the HIGH/CRITICAL CVEs with the fix path (direct vs transitive, target version) without creating a file.

**Do not** stack intermediate questions ("Do you confirm?", "On which branch?", "Do you want the commit?"). Ancillary decisions are made **either** from the CLI options **or** in a single block of questions at the end.

### 6. Verification

After any dependency change:

1. **PHP**: run the project's test suite (detected via `composer.json` or `CLAUDE.md`)
   - Pest: `dock php ./vendor/bin/pest` (or equivalent)
   - PHPUnit: `dock php ./vendor/bin/phpunit`
2. **JS**: run the tests if a `test` script is defined in `package.json`
3. **Lint / build**: run them if available (`composer analyse`, `npm run build`)

On failure: diagnose, revert if necessary, inform the user.

### 7. Final summary

Present a recap **only if an action was performed** (save, update, workplan). Otherwise the console display is enough, no extra noise.

If an action was performed:

```
✅ Report saved: docs/audits/packages-2026-04-15.md
✅ 5 minor updates applied (composer update)
✅ Tests passed: 248/248

📋 To address next:
  🟡 3 medium not addressed
  🔴 2 critical not addressed
  🔐 1 HIGH CVE not covered
```

## Special cases

### Monorepo / multi-workspace project

- If `pnpm-workspace.yaml` or `packages/*/package.json` are detected, run the audit on each workspace
- Aggregate the results by grouping the same packages but noting the workspaces involved

### Dependencies not tracked by `composer outdated`

Packages on `dev-main`, `dev-<branch>` or with `@dev` constraints do not necessarily show up in `outdated`. Mention them explicitly as "not automatically assessable".

### Project without Docker

If `CLAUDE.md` does not indicate a Docker wrapper, try the commands locally (`composer`, `npm`). On failure, ask the user for the execution strategy.

### Deprecated packages

When `composer audit` or `npm audit` flags an **abandoned** package (`abandoned`), automatically classify it as 🔴 Critical even if no vulnerability is associated: migration is unavoidable.

### No update available

If all packages are up to date: generate a minimal "nothing to report" type report, without creating a markdown file (unless `--save` is explicit).

## Additional references

The detailed classification criteria and the feasibility grid are described in:

- `references/classification.md` — full grid of minor/medium/critical criteria
- `references/commands.md` — cheatsheet of commands per manager

## Usage examples

```bash
# Full PHP + JS audit, console display + a single grouped question at the end
/audit-packages

# PHP-only audit
/audit-packages --scope=php

# Audit + automatic markdown save (no question at the end)
/audit-packages --auto-save

# Quietest possible audit (useful for CI / pipelines)
/audit-packages --quiet --auto-save
```
