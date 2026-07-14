# Audit commands cheatsheet

Quick reference of commands per package manager. Consult it if there is any doubt about the syntax or useful options.

All the commands below must be **adapted to the project environment** (`dock` wrapper, `docker compose exec`, direct execution, etc.). Consult `CLAUDE.md` to know the execution strategy.

## Composer (PHP)

### List outdated packages

```bash
# Direct dependencies only (recommended for the audit)
composer outdated --direct --format=json

# All dependencies (including transitive)
composer outdated --format=json

# Human-readable display
composer outdated --direct
```

### Security audit (Composer >= 2.4)

```bash
composer audit --format=json
```

Output: a list of `advisories` with `cve`, `severity`, `title`, `link`, `affectedVersions`, `reportedAt`.

### Inspect a specific package

```bash
composer show vendor/package
composer show vendor/package --all  # all available versions
```

### Test an update before applying it

```bash
composer update vendor/package --dry-run --with-dependencies
```

## npm (JS)

### List outdated packages

```bash
npm outdated --json
npm outdated --long  # readable display
```

JSON output per package: `current`, `wanted` (satisfies the manifest's semver constraint), `latest`, `dependent`, `location`.

### Security audit

```bash
npm audit --json
npm audit --omit=dev --json  # ignore devDependencies
```

Output: a `vulnerabilities` object with details per package, and `metadata.vulnerabilities` aggregated by severity.

### Apply automatic fixes

```bash
npm audit fix           # respects semver
npm audit fix --force   # includes breaking changes (avoid without validation)
```

## Yarn v1 (Classic)

```bash
yarn outdated --json
yarn audit --json
```

## Yarn Berry (v2+)

```bash
yarn npm audit --json
yarn npm audit --recursive --json  # workspaces
```

To list outdated packages, installing the `outdated` plugin may be necessary:

```bash
yarn plugin import https://go.mskr.ca/yp-outdated
yarn outdated
```

## pnpm

```bash
pnpm outdated --format json
pnpm outdated --recursive --format json  # workspaces
pnpm audit --json
pnpm audit --prod --json  # ignore devDependencies
```

## JS manager detection

In order of priority:

| File present | Manager |
|---|---|
| `pnpm-lock.yaml` | pnpm |
| `yarn.lock` + `.yarnrc.yml` | yarn berry |
| `yarn.lock` (without `.yarnrc.yml`) | yarn v1 |
| `package-lock.json` | npm |
| no lock | npm (default) |

Also check the `packageManager` field in `package.json` (if present, it takes priority).

## Apply updates

### Composer

```bash
# Minor / patch (respects the composer.json constraints)
composer update --with-dependencies vendor/pkg1 vendor/pkg2

# Force a major version (updates the constraint)
composer require vendor/package:^2.0
```

### npm

```bash
npm update pkg1 pkg2                  # respects the constraint
npm install pkg@latest                 # forces the latest
```

### yarn v1

```bash
yarn upgrade pkg --latest
```

### yarn berry

```bash
yarn up pkg            # respects the constraint
yarn up pkg@^2         # updates the constraint
```

### pnpm

```bash
pnpm update pkg        # respects the constraint
pnpm update pkg --latest  # forces the latest
```

## Parsing JSON output

The `--json` / `--format=json` format varies across tools. A few tips:

- **npm outdated --json**: object `{ "package-name": { current, wanted, latest, ... } }` — may be empty if everything is up to date
- **composer outdated --format=json**: `installed` key containing an array of objects
- **composer audit --format=json**: `advisories` key containing an object indexed by advisory ID
- **pnpm outdated --format json**: object indexed by package name

Use `jq` to extract the relevant information, e.g.:

```bash
composer outdated --direct --format=json | jq '.installed[] | {name, version, latest, "latest-status"}'
```

## Commands to retrieve additional info

### Changelog / release notes

- **Composer**: `composer show vendor/package --all` then `WebFetch` on Packagist or the GitHub release
- **npm**: `npm view package versions`, `npm view package repository.url`

### Usage footprint of a package in the code

```bash
# PHP (namespace-based usage)
grep -r "use Vendor\\\\Package" src/ app/ --include="*.php" | wc -l

# JS (import-based usage)
grep -r "from ['\"]package-name" src/ --include="*.{ts,tsx,js,jsx,vue}" | wc -l
```

Use `Agent(explore-codebase)` for a richer assessment (usage patterns, dynamic imports, etc.).
