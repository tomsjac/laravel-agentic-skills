---
name: fix-tests-php
description: "Analyzes and fixes failing PHP/Laravel tests (PHPUnit, Pest)"
argument-hint: "[--file=PATH] [--filter=TEST_NAME] [--dry-run] [--fix-code]"
allowed-tools: Bash(php *), Bash(dock *), Bash(docker *), Bash(git *), Read, Edit, Glob, Grep, Agent(explore-codebase), Agent(explore-tests)
accept-on: Edit
---

# Fix Tests PHP

## Output language

Produce all user-facing content (messages, generated documents, titles, descriptions, commit messages, changelog, summaries) in the user's language: match the language the user writes in (French or English). If undetermined, default to English. These instructions are written in English and do not dictate the output language.

Analyze and fix failing PHP tests: **$ARGUMENTS**

## Objective

Run the project's PHP tests (PHPUnit or Pest), analyze the failures, determine whether the problem comes from the **test** or the **source code**, and apply the appropriate fix.

## Options

- `--file=PATH`: Run only a specific test file
- `--filter=TEST_NAME`: Filter on a specific test name
- `--dry-run`: Analyze the failures without fixing them
- `--fix-code`: Force fixing the source code (by default the skill automatically determines whether to fix the test or the code)

## Workflow

### 1. Test framework detection

Identify the framework in use:
- `phpunit.xml` or `phpunit.xml.dist` → PHPUnit
- `pest.php` or the `pestphp/pest` dependency in `composer.json` → Pest
- Check the version (PHPUnit 10+, Pest 2+, etc.)

Read the configuration to understand:
- The test suites (Feature, Unit, etc.)
- The bootstrap
- The extensions/plugins

### 2. Running the tests

Check the project's `CLAUDE.md`, `AGENTS.md`, or `README.md` to determine how to run PHP commands (Docker, the `dock` wrapper, direct execution, etc.). Adapt the commands below accordingly.

**Without options** (all tests):
```bash
php ./vendor/bin/phpunit --no-coverage 2>&1
```

**With `--file`**:
```bash
php ./vendor/bin/phpunit --no-coverage <PATH> 2>&1
```

**With `--filter`**:
```bash
php ./vendor/bin/phpunit --no-coverage --filter <TEST_NAME> 2>&1
```

Capture the full output (stdout + stderr).

### 3. Parsing the results

Parse the output to extract:
- Total number of tests, assertions, failures, errors, skipped
- For each failure/error:
  - Test name (class + method)
  - File and line
  - Error type (assertion failed, exception, error)
  - Full error message
  - Expected vs actual diff (if an assertion)

Display a summary:
```
Test results:

  Tests: 142 | Passed: 138 | Failed: 3 | Errors: 1

  Failures:
  1. ❌ Tests\Feature\UserControllerTest::test_user_can_update_profile
     → assertStatus(200) failed, got 422
     → tests/Feature/UserControllerTest.php:87

  2. ❌ Tests\Feature\SurveyServiceTest::test_create_survey_with_null_date
     → TypeError: Carbon\Carbon::parse(): Argument #1 must be string, null given
     → app/Services/SurveyService.php:92

  3. ❌ Tests\Unit\DateHelperTest::test_format_date_french
     → assertEquals failed: expected "13 mars 2026", got "13 March 2026"
     → tests/Unit/DateHelperTest.php:34

  4. 💥 Tests\Feature\ApiAuthTest::test_login_with_invalid_credentials
     → Illuminate\Database\QueryException: SQLSTATE[42S02] Table not found
     → app/Models/User.php:15
```

If `--dry-run` → stop here.

### 4. Analyzing each failure

For each failing test, determine the source of the problem:

**The problem comes from the TEST if**:
- The test is obsolete (it tests a behavior that intentionally changed)
- The test data is incorrect (Factory, fixtures)
- The assertions are poorly written or too strict
- The mock/stub is misconfigured
- The test does not reflect the new specs

**The problem comes from the SOURCE CODE if**:
- The test is correct but the code does not do what it should
- A regression was introduced
- A TypeError or an unhandled exception
- An unexpected behavior

**Analysis method**:
1. Read the full test to understand the expected behavior
2. Read the source code under test
3. Compare the expected vs actual behavior
4. Check the recent changes (`git log --oneline -10 -- <file>`)

Present the diagnosis for each failure:
```
Diagnosis:

1. test_user_can_update_profile → 🔧 Fix CODE
   The FormRequest added a required 'phone' validation,
   but the test does not provide that field.
   → Fix the test (add 'phone' to the data)

2. test_create_survey_with_null_date → 🔧 Fix CODE
   SurveyService::create() does not handle ended_at = null
   → Fix the source code (add null check)
```

If `--fix-code` is provided, always fix the source code rather than the test.

### 5. Fixing

For each failure, apply the identified fix:

**If fixing the test**:
- Update the test data (Factory states, manual data)
- Fix the assertions
- Update the mocks/stubs
- Follow the project's AAA pattern (Arrange, Act, Assert)

**If fixing the source code**:
- Apply the minimal fix that makes the test pass
- Do not change the behavior beyond what the test requires
- Check consistency with the other tests

### 6. Verification

Re-run the fixed tests to confirm:

```bash
php ./vendor/bin/phpunit --no-coverage --filter "test_user_can_update_profile|test_create_survey" 2>&1
```

If any tests still fail, iterate (max 3 attempts).

### 7. Final summary

```
Summary of fixes:

  Before: 4 failures out of 142 tests
  After: 0 failures out of 142 tests ✅

  Fixes applied:
  ✅ test_user_can_update_profile → Test updated (added phone field)
  ✅ test_create_survey_with_null_date → Source code fixed (null check)
  ✅ test_format_date_french → Test updated (Carbon locale)
  ✅ test_login_with_invalid_credentials → Source code fixed (missing migration)

  Modified files:
    - tests/Feature/UserControllerTest.php
    - app/Services/SurveyService.php
    - tests/Unit/DateHelperTest.php
    - database/migrations/2026_03_13_create_sessions_table.php
```

## Usage Examples

```bash
# Run all tests and fix the failures
/fix-tests-php

# Fix a specific test file
/fix-tests-php --file=tests/Feature/UserControllerTest.php

# Fix a specific test
/fix-tests-php --filter=test_user_can_update_profile

# View the failures without fixing
/fix-tests-php --dry-run

# Force fixing the source code (not the tests)
/fix-tests-php --fix-code
```
