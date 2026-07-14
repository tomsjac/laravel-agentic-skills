---
name: write-tests
description: "Creates functional PHP/Laravel tests following the AAA pattern (Arrange, Act, Assert). Use to write Feature tests, validate business behavior, or test API endpoints."
argument-hint: "[test description or file/class to test] [--unit] [--coverage]"
allowed-tools: Bash(php *), Bash(git diff:*), Bash(git status:*), Agent(explore-codebase), Agent(explore-tests), Agent(explore-db)
---

# Creating Functional PHP/Laravel Tests

## Output language

Produce all user-facing content (messages, generated documents, titles, descriptions, commit messages, changelog, summaries) in the user's language: match the language the user writes in (French or English). If undetermined, default to English. These instructions are written in English and do not dictate the output language.

Create tests for: $ARGUMENTS

## Project Context

- PHP stack: !`php -r "echo PHP_VERSION;" 2>/dev/null || echo "inconnu"`
- PHPUnit: !`php vendor/bin/phpunit --version 2>/dev/null | head -1 || echo "inconnu"`
- Branch: !`git branch --show-current`
- Recent changes: !`git diff --name-only HEAD~3 2>/dev/null | grep -E '\.(php)$' | head -20`

## Core Principles

### Mandatory AAA pattern

Each test MUST strictly follow the 3 phases:

```php
public function test_clear_description_of_the_behavior(): void
{
    // Arrange — set up the data and context
    $user = User::factory()->create(['role' => 'admin']);
    $post = Post::factory()->create(['status' => 'draft']);

    // Act — perform A SINGLE action
    $response = $this->actingAs($user)->put("/api/posts/{$post->id}", [
        'status' => 'published',
    ]);

    // Assert — verify the results
    $response->assertOk();
    $this->assertDatabaseHas('posts', [
        'id' => $post->id,
        'status' => 'published',
    ]);
}
```

### Strict rules

1. **Prioritize Feature tests** — test the application's real behavior over HTTP
2. **Avoid mocks** — use the real database and real classes. Mock ONLY external services (third-party APIs, email sending, remote file systems)
3. **One behavior per test** — a single action in the Act phase
4. **Explicit naming** — `test_<subject>_<action>_<expected_result>` in snake_case
5. **Independent tests** — no test depends on another, random execution order
6. **Specific assertions** — use `assertSame` rather than `assertEquals`, check the database rather than just the HTTP response

### When to use `--unit`

Switch to a unit test only for:
- Pure business logic with no Laravel dependency (Value Objects, calculation Services)
- Helpers and utilities
- Data transformations

## Workflow

### 1. Preliminary exploration

Before writing a single test:

1. **Read the code to test** — understand the logic, dependencies, edge cases
2. **Identify existing tests** — check `tests/Feature/` and `tests/Unit/` for the project's patterns
3. **Analyze the structure** — routes, middleware, form requests, policies, observers
4. **Check the factories** — make sure the required factories exist in `database/factories/`

Use the `explore-codebase`, `explore-tests` and `explore-db` agents for this phase.

### 2. Test planning

List the test cases to cover:

| Case | Type | Priority |
|-----|------|----------|
| Nominal case (happy path) | Feature | High |
| Input validation | Feature | High |
| Authorizations / Policies | Feature | High |
| Edge cases | Feature | Medium |
| Expected errors | Feature | Medium |
| Pure business logic | Unit (if `--unit`) | Variable |

Present this list to the user for validation before coding.

### 3. Creating the tests

Conventions to follow:

- **Location**: `tests/Feature/` for functional tests, `tests/Unit/` for unit tests
- **File naming**: `<Sujet>Test.php` (e.g. `PostPublicationTest.php`)
- **Namespace**: matching the directory (`Tests\Feature\...` or `Tests\Unit\...`)
- **Class**: `final class`, `extends TestCase`
- **Trait**: `use RefreshDatabase;` for Feature tests with a database
- **Methods**: `test_` prefix in snake_case, `void` return type

Typical file structure:

```php
<?php

declare(strict_types=1);

namespace Tests\Feature;

use App\Models\User;
use Illuminate\Foundation\Testing\RefreshDatabase;
use Tests\TestCase;

final class ExempleTest extends TestCase
{
    use RefreshDatabase;

    public function test_cas_nominal(): void
    {
        // Arrange
        // Act
        // Assert
    }

    public function test_cas_erreur(): void
    {
        // Arrange
        // Act
        // Assert
    }
}
```

See [best-practices.md](references/best-practices.md) for detailed rules on assertions, factories and mocks.
See [examples.md](references/examples.md) for concrete examples by test type.

### 4. Execution and validation

Run the tests. Adapt the command to the project's conventions (Docker, wrapper, direct execution, etc.) by checking the project's `CLAUDE.md` or `README.md`:

```bash
# Run the created tests
php vendor/bin/phpunit tests/Feature/TestName.php

# With coverage if --coverage
php vendor/bin/phpunit tests/Feature/TestName.php --coverage-text
```

If a test fails:
1. Analyze the error message
2. Check that the application code is correct (the test may reveal a real bug)
3. Fix the test only if the expected behavior is confirmed
4. Re-run until all tests pass

### 5. Final check

- [ ] All tests pass
- [ ] Each test follows the AAA pattern with section comments
- [ ] Explicit naming describing the tested behavior
- [ ] No unnecessary mocks (external services only)
- [ ] Factories used for test data
- [ ] Specific and relevant assertions
- [ ] Tests independent of one another
</content>
</invoke>
