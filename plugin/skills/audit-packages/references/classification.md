# Update classification — detailed criteria

This document complements the `audit-packages` `SKILL.md` and serves as a reference when an ambiguous case arises.

## Three axes of analysis

Classifying an outdated package combines **three** independent axes. A package is classified at the **highest level** reached on any of these axes.

1. **Semver jump type** — theoretical degree of risk based on the version numbering
2. **Position of the package in the architecture** — role played in the project
3. **Security state** — known vulnerabilities on the installed version

## Axis 1 — Semver jump type

| Observed jump | Default level |
|---|---|
| patch (`1.2.3` → `1.2.4`) | 🟢 Minor |
| minor (`1.2.3` → `1.3.0`) | 🟢 Minor |
| major (`1.2.3` → `2.0.0`) | 🔴 Critical |
| pre-1.0 minor (`0.4.x` → `0.5.0`) | 🟡 Medium (pre-1.0 semver allows breaking changes in a minor) |
| moving from `v1.x.x-dev` to stable | 🟡 Medium |

**Exceptions to bump up one notch**:
- A minor jump that crosses a boundary documented as major by the maintainer (e.g. Laravel 10.x → 10.48.x including an internal API change)
- A package whose maintainers do not strictly follow semver (to identify via changelog history)

## Axis 2 — Position of the package in the architecture

| Package role | Examples | Minimum level |
|---|---|---|
| Framework | Laravel, Symfony, Vue, React, Nuxt | 🟡 Medium even for a minor |
| ORM / database | Eloquent, Doctrine, Prisma, TypeORM | 🟡 Medium |
| Runtime / compiler | PHP itself (via composer constraint), TypeScript, Vite | 🟡 Medium |
| Security / auth | Sanctum, Passport, Passkit, JWT libs | 🟡 Medium |
| Payment / billing | Stripe, PayPal, billing lib | 🟡 Medium |
| Test / dev-only | Pest, PHPUnit, Vitest, ESLint | 🟢 Minor (except major) |
| Targeted utility | lodash, date-fns, internal helpers | 🟢 Minor (except major) |

**Principle**: the more central the package, the more diffuse the consequences of a change, even for a minor jump. A minor jump of Laravel warrants a review; a minor jump of a utility lib can be done blindly.

## Axis 3 — Security state

| Severity (CVSS or manager label) | Imposed level |
|---|---|
| CRITICAL | 🔴 Critical (address immediately) |
| HIGH | 🔴 Critical |
| MEDIUM / MODERATE | 🟡 Medium |
| LOW | 🟢 Minor (but not to be ignored) |
| INFO / none | — (imposes nothing) |

**Special case: abandoned package**:
- `composer audit` may report `abandoned` with a replacement recommendation
- `npm` may report `deprecated`
- Always 🔴 Critical: migration is unavoidable eventually, so better to plan it

## Feasibility grid

This grid does not impose a level, it **informs** the user about the estimated cost:

| Criterion | Easy | Moderate | Complex |
|---|---|---|---|
| Files using the package (via grep) | < 5 | 5 – 20 | > 20 |
| Breaking changes in the changelog | none | a few, isolated | many or cross-cutting |
| Tests covering these files | > 80% | 30 – 80% | < 30% or unknown |
| Package used in critical files (routes, core business) | no | partially | yes |
| Migration tool provided by the maintainer (codemod, rector, eslint rule) | yes | partial | no |

**Estimation method**:
1. Identify the package's **import name** (e.g. `guzzlehttp/guzzle` → `use GuzzleHttp\\`)
2. Count via grep: `grep -r "use GuzzleHttp" src/ | wc -l`
3. Examine 2-3 files at random to estimate the migration complexity
4. Read the "Breaking changes" section of the official changelog

## Ambiguous cases and decision rules

### A package has both a minor and a major version available

Priority: propose the **minor version** first (safe), then flag the major as an upgrade to plan separately.

### A vulnerability is fixed only in a major version

The package becomes 🔴 Critical (the major must be migrated), but the vulnerability must be noted as a **time constraint**.

### A package has had no updates for > 2 years

Flag it as **at risk** in the report, even if it does not appear in `outdated`. Check whether it has been replaced by an active fork.

### Restrictive constraint blocking an update

Example: `"php": "^8.1"` blocks a package requiring `^8.2`. Do not automatically classify as critical: mention the blocking constraint in the report and let the user decide whether the project constraint should evolve.

### Multiple packages sharing a vulnerable transitive dependency

If `composer audit` or `npm audit` reports a CVE on a transitive dependency, identify the direct package(s) that introduce it and propose the update at the direct package level (more reliable than a forced `composer require` on the transitive).

### Specific handling of vulnerable transitive dependencies

Transitive dependencies deserve their own line in the report, with the **Via** column indicating the responsible direct package.

Two scenarios:

1. **The direct package has an available update that fixes the transitive**:
   - Classification level = the direct package's level (the transitive CVE becomes an **argument to prioritize** the direct upgrade, not an independent level)
   - E.g. `chokidar@2 → chokidar@5` simultaneously fixes `anymatch`, `braces`, `readdirp`

2. **The direct package is internalized / frozen / without an update** (internal fork, version pinned by line `"kateji/framework": "4.1.14"`):
   - The problem cannot be solved by a standard `composer update` command
   - Classify as 🔴 Critical with the note **"blocked by <direct package> — maintainer coordination required"**
   - Do not propose an automatic update: the user must decide how to unblock (fork, patch via forced `composer require`, contact the maintainer)

### "wanted == current" case (npm)

When `npm outdated` returns `wanted == current`, it means the `package.json` constraint (e.g. `"chokidar": "^2.1.8"`) blocks any update, even a minor patch. The available jump is necessarily **major**.

Classification impact:
- The package is automatically classified as 🔴 Critical (a major jump is required to unblock)
- In the report, explicitly flag the **blocking constraint** so the user understands that simply running `npm update` will not suffice — the `package.json` must be edited

This case is **very common** in old, unmaintained projects: the entire dependency chain becomes 🔴 Critical simultaneously. In this case present an honest summary ("100% of the packages are frozen by their constraints") rather than trying to absorb it into the standard grid.

### Blocking runtime constraint (e.g. PHP 7.3)

If `composer.json` contains a `"php": "^7.3"` constraint but a package update requires PHP 8.1+, mention the **blocking constraint** in the report:

> `monolog/monolog` 3.x requires PHP 8.1+ (project constraint: `^7.3`). Update impossible without raising the project's PHP constraint.

Do not automatically classify as Critical on this criterion alone: it is a **project constraint** that only the user can decide. But highlight it in the observations, because it may represent a broader modernization effort.

## Classification output format

For each audited package, produce the following conceptual object (useful for report generation):

```
{
  "name": "guzzlehttp/guzzle",
  "ecosystem": "php",
  "current": "7.8.0",
  "minor": "7.9.2",
  "major": "8.0.0",
  "level": "critical",
  "feasibility": "complex",
  "signals": [
    "major-jump",
    "files-impacted: 34",
    "breaking-changes-documented"
  ],
  "vulnerabilities": [],
  "changelog_url": "https://github.com/guzzle/guzzle/releases/tag/8.0.0",
  "recommended_action": "workplan"
}
```

This schema is **not** a required deliverable, it is there to frame the thinking during analysis.
