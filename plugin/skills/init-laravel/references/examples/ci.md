# CI/CD - Example files

**Generic, self-contained** CI/CD configuration examples for Laravel projects.
Adapt the `{{PHP_VERSION}}` (e.g. `8.4`) and `{{DB_VERSION}}` (e.g. `11`) placeholders to the project.

The marketplace is multi-forge, but projects are most likely hosted on **GitHub**, so **GitHub Actions is the default** and `.gitlab-ci.yml` is **not** created unless the project explicitly lives on GitLab.

## .github/workflows/ci.yml (GitHub Actions — default)

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:

jobs:
  tests:
    runs-on: ubuntu-latest
    services:
      mariadb:
        image: mariadb:{{DB_VERSION}}
        env:
          MYSQL_DATABASE: testing
          MYSQL_ROOT_PASSWORD: secret
        ports:
          - 3306:3306
        options: >-
          --health-cmd="healthcheck.sh --connect --innodb_initialized"
          --health-interval=10s --health-timeout=5s --health-retries=5
    steps:
      - uses: actions/checkout@v4
      - uses: shivammathur/setup-php@v2
        with:
          php-version: "{{PHP_VERSION}}"
          extensions: zip, pdo_mysql
          coverage: none
      - run: composer install --no-interaction --prefer-dist --no-progress
      - name: Pint
        run: vendor/bin/pint --test
      - name: PHPStan
        run: vendor/bin/phpstan analyse --no-progress
        continue-on-error: true
      - name: Tests
        env:
          DB_HOST: 127.0.0.1
          DB_DATABASE: testing
          DB_USERNAME: root
          DB_PASSWORD: secret
        run: |
          cp .env.example .env
          php artisan key:generate
          php artisan test
```

## .gitlab-ci.yml (GitLab CI — only if hosted on GitLab)

```yaml
stages:
  - quality
  - test

variables:
  COMPOSER_ALLOW_SUPERUSER: "1"

default:
  image: php:{{PHP_VERSION}}-cli
  cache:
    key: composer-cache
    paths:
      - vendor/
  before_script:
    - apt-get update && apt-get install -y git unzip libzip-dev && docker-php-ext-install zip pdo_mysql
    - curl -sS https://getcomposer.org/installer | php -- --install-dir=/usr/local/bin --filename=composer
    - composer install --no-interaction --prefer-dist --no-progress

pint:
  stage: quality
  script:
    - vendor/bin/pint --test

phpstan:
  stage: quality
  script:
    - vendor/bin/phpstan analyse --no-progress
  allow_failure: true

tests:
  stage: test
  services:
    - name: mariadb:{{DB_VERSION}}
      alias: mariadb
  variables:
    MYSQL_DATABASE: testing
    MYSQL_ROOT_PASSWORD: secret
    DB_HOST: mariadb
    DB_DATABASE: testing
    DB_USERNAME: root
    DB_PASSWORD: secret
  script:
    - cp .env.example .env
    - php artisan key:generate
    - php artisan test
```

## Pull/Merge Request template

Generic example. Place it in `.github/pull_request_template.md` for GitHub (default),
or `.gitlab/merge_request_templates/Default.md` for GitLab.

```markdown
## Related issue

- **Link:**

## Description

<!-- Briefly describe what was done -->

| Criterion | Value |
|---------|--------|
| Change type | Feature / Bugfix / Refactor / Chore |
| Impact level | Low / Medium / High |

## Post-deployment actions

- [ ] No action required

## Review checklist

- [ ] The code follows the project conventions
- [ ] The tests pass
- [ ] Static analysis does not report any new errors
- [ ] No obvious regression
- [ ] Documentation is up to date if needed
```
