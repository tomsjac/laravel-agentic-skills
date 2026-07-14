---
name: fix-phpstan
description: "Automatically analyzes and fixes PHPStan errors in the Laravel project"
argument-hint: "[--level=N] [--file=PATH] [--dry-run] [--memory=2G]"
allowed-tools: Bash(php *), Bash(dock *), Bash(docker *), Bash(git *), Read, Edit, Glob, Grep, Agent(explore-codebase)
accept-on: Edit
---

# Fix PHPStan

## Output language

Produce all user-facing content (messages, generated documents, titles, descriptions, commit messages, changelog, summaries) in the user's language: match the language the user writes in (French or English). If undetermined, default to English. These instructions are written in English and do not dictate the output language.

Analyze and fix PHPStan errors: **$ARGUMENTS**

## Goal

Run PHPStan inside the project's Docker container, parse the errors, and fix them automatically in batches while following the project's conventions.

## Current Context

- Current branch: !`git branch --show-current`
- Modified files: !`git status --porcelain`

## Options

- `--level=N`: Force a specific PHPStan level (otherwise uses the project config)
- `--file=PATH`: Analyze only a specific file or directory
- `--dry-run`: Display the errors without fixing them
- `--memory=SIZE`: Memory limit for PHPStan (default: 2G)

## Workflow

### 1. Detecting the PHPStan configuration

Look for the project's PHPStan configuration:
- `phpstan.neon` or `phpstan.neon.dist` at the root
- `phpstan` section in `composer.json`

Read the configuration to understand:
- The configured analysis level
- The analyzed paths
- The ignored errors (baseline, ignoreErrors)
- The loaded extensions (larastan, etc.)

If no configuration is found, inform the user and offer to create one.

### 2. Running PHPStan

Check the project's `CLAUDE.md`, `AGENTS.md`, or `README.md` to determine how to run PHP commands (Docker, the `dock` wrapper, direct execution, etc.). Adapt the commands below accordingly.

```bash
php ./vendor/bin/phpstan analyse --memory-limit=2G --no-progress --error-format=json
```

If `--file` is provided:
```bash
php ./vendor/bin/phpstan analyse <PATH> --memory-limit=2G --no-progress --error-format=json
```

If `--level` is provided, add `--level=N`.

Capture the full JSON output.

### 3. Parsing and categorizing the errors

Parse the PHPStan JSON and group the errors by category:

| Category | PHPStan identifier examples |
|---|---|
| **Missing types** | `missingType.return`, `missingType.parameter`, `missingType.property`, `missingType.iterableValue` |
| **Return types** | `return.type`, `return.missing`, `return.void` |
| **Parameters** | `argument.type`, `parameter.type` |
| **Properties** | `property.notFound`, `property.nonObject`, `property.readOnly` |
| **Methods** | `method.notFound`, `method.nonObject`, `staticMethod.notFound` |
| **Variables** | `variable.undefined`, `variable.certainty` |
| **Dead code** | `deadCode.unreachable`, `if.alwaysTrue`, `if.alwaysFalse` |
| **Other** | Everything else |

Display a summary:
```
PHPStan error summary:

  12 × Missing types (return types, param types)
   5 × Properties not found
   3 × Methods not found
   2 × Dead code
  ──
  22 errors total across 15 files
```

If `--dry-run` → stop here.

### 4. Batch fixing

Fix the errors **by category**, starting with the simplest and most numerous ones:

**Recommended fixing order**:
1. **Missing types**: Add return types, param types, property types by analyzing the code
2. **Dead code**: Remove the identified dead code
3. **Incorrect return types**: Fix the return types
4. **Properties/methods**: Check whether it's a real bug or a false positive (IDE helper, Laravel macros, etc.)

**For each file to fix**:
1. Read the entire file
2. Understand the context (class, methods, inheritance)
3. Apply the appropriate fix
4. Don't break existing behavior

**Fixing rules**:
- Use native PHP types whenever possible (`string`, `int`, `bool`, `array`, `?Type`)
- Use PHPDoc generics for collections (`@param Collection<int, Model>`)
- Follow the project's existing conventions (PHPDoc vs native type hints)
- Don't fix errors coming from `vendor/` or external packages
- If an error is a known false positive (e.g. Laravel macros, IDE helper), add a `@phpstan-ignore` with the reason

### 5. Verification

Re-run PHPStan after the fixes:

```bash
php ./vendor/bin/phpstan analyse --memory-limit=2G --no-progress --error-format=json
```

Compare with the initial result.

### 6. Final summary

```
PHPStan fix summary:

  Before: 22 errors across 15 files
  After: 3 errors across 2 files

  Fixed:
  ✅ 12 missing types added
  ✅ 5 properties fixed
  ✅ 2 dead code removed

  Remaining (false positives or manual fixes needed):
  ⚠️ 3 errors across 2 files (details below)
    - app/Services/FooService.php:42 → method.notFound (Laravel macro)
    - app/Models/Bar.php:15 → property.notFound (dynamic relation)
    - app/Models/Bar.php:28 → return.type (complex union type)
```

## Handling special cases

### Laravel / Larastan
- Eloquent relations (`hasMany`, `belongsTo`) often require specific `@return` annotations
- Dynamic macros and scopes can produce false positives → use `@phpstan-ignore`
- Check whether `_ide_helper.php` is up to date (`php artisan ide-helper:generate`)

### Baseline
- If the project uses a baseline (`phpstan-baseline.neon`), only fix the **new errors** (not the ones in the baseline)
- Offer to regenerate the baseline if many errors remain

## Usage Examples

```bash
# Fix all PHPStan errors
/fix-phpstan

# Analyze only one file
/fix-phpstan --file=app/Services/UserService.php

# See the errors without fixing them
/fix-phpstan --dry-run

# Force level 8
/fix-phpstan --level=8
```
