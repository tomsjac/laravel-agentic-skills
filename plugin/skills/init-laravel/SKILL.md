---
name: init-laravel
description: "Initialize a new Laravel project with a complete configuration (monolith or API/microservice)"
argument-hint: "[--monolith] [--api] [--microservice] [--name=NAME]"
allowed-tools: Bash(*), Read, Write, Edit, Glob, Grep, Agent(explore-codebase), Agent(web-search)
---

# Setup Project Laravel

## Output language

Produce all user-facing content (messages, generated documents, titles, descriptions, commit messages, changelog, summaries) in the user's language: match the language the user writes in (French or English). If undetermined, default to English. These instructions are written in English and do not dictate the output language.

Initialize a new Laravel project: **$ARGUMENTS**

## Goal

Create a new Laravel project with a complete, ready-to-use configuration, tailored to the project type (monolith with frontend, pure API, or microservice).

## Principle: prefer first-party Laravel tooling

This skill relies on the official Laravel tooling rather than reinventing it:

- **`laravel new`** (the Laravel installer) for project creation — it interactively scaffolds the starter kit (frontend), the testing framework, and the database.
- **Laravel Sail** for the local Docker environment (default).
- **Laravel Boost / PAO / AI SDK** for optional AI integration.

Reference: [Laravel installation docs](https://laravel.com/docs/13.x/installation).

## Project types

### `--monolith`: Monolithic application (frontend + backend)

A full Laravel project with an integrated user interface. Two scaffolding paths (detailed in step 2): the **official starter kit** (`composer create-project laravel/<vue|react|livewire>-starter-kit` — clean build, pins Laravel 12) or **Breeze** (`laravel new --breeze --stack=…` — Laravel 13, needs frontend fixes).

**Ask the user which frontend stack they want:**
1. **Livewire** (Blade + Livewire) — `laravel/livewire-starter-kit` or `--breeze --stack=livewire`
2. **Vue.js + Inertia** — `laravel/vue-starter-kit` or `--breeze --stack=vue`
3. **React + Inertia** — `laravel/react-starter-kit` or `--breeze --stack=react`
4. **Blade only** — `--breeze --stack=blade`

> Avoid `--typescript` with Breeze on Laravel 13 (incomplete scaffold — see step 2 caveat). For teams / 2FA / API scaffolding, use Jetstream (`--jet`) instead.

### `--api`: REST API

A Laravel project configured to expose a REST API only. No frontend, no views, no sessions.

### `--microservice`: Microservice

A minimal Laravel project optimized for a microservice. Removes everything unnecessary (views, sessions, cookies, CSRF, etc.).

**If no type is provided**, ask the user:
```
Which type of Laravel project?

1. Monolith (frontend + backend)
2. REST API (backend only)
3. Microservice (minimal backend)

Choice:
```

## Options

- `--monolith`: Monolithic application with frontend
- `--api`: REST API only
- `--microservice`: Minimal microservice
- `--name=NAME`: Project name (otherwise asked interactively)

## Workflow

### 1. Gather information

Ask for the required information:
- **Project name** (if `--name` is not provided)
- **Project type** (if no type option is provided)
- **Frontend stack** (if `--monolith`: Livewire, Vue+Inertia, React+Inertia, Blade-only) — drives the starter kit
- **Database** — **default to MySQL/MariaDB** for monolith and API projects (`--database=mysql`); offer PostgreSQL/SQLite only if the user asks. `laravel new` also prompts for this. Because the DB runs in Docker (Sail), set up Sail **before** any migration (see step 4 ordering rule).
- **Testing framework** (Pest or PHPUnit) — `laravel new` prompts for this

> The Laravel installer asks for the starter kit, the database, and the testing framework interactively. Pass them as flags for a non-interactive run when the answers are known.

### 2. Install Laravel (official installer)

If the `laravel` installer is not available, install it first:

```bash
composer global require laravel/installer
```

Create the project with `laravel new`. **Everything below is non-interactive — the agent runs it directly, never delegate to the user.** Always pass `--no-interaction`, otherwise the installer shows a prompt (e.g. *"Would you like to install a starter kit?"*) that, without a TTY, silently falls back to *no* starter kit.

- **Monolith** — two paths, pick based on the Laravel version you need:

  **Option A (recommended for a hassle-free frontend) — official starter kit via `composer create-project`.** Self-contained, TypeScript-native, modern UI (shadcn-style components, headless-ui, lucide), and **`npm run build` works out of the box with zero fixes**. Trade-off: the kit currently **pins Laravel 12** (not 13).

  ```bash
  # Pick the kit matching the frontend choice:
  #   laravel/vue-starter-kit | laravel/react-starter-kit | laravel/livewire-starter-kit
  composer create-project laravel/vue-starter-kit <NAME> --no-interaction
  cd <NAME>
  npm install && npm run build
  ```

  These kits default to SQLite and run migrations on create. Afterwards switch to **MySQL** in `.env` (`DB_CONNECTION=mysql` + `DB_*`) and re-migrate against Sail (see step 4 ordering rule).

  **Option B — `laravel new --breeze` (gets Laravel 13, but the frontend needs reconciling).** This installer's starter-kit prompt only offers **Breeze**/**Jetstream** (not the standalone kits), driven non-interactively by `--breeze`/`--jet` + `--stack`:

  ```bash
  # --stack=blade | livewire | vue | react | api
  laravel new <NAME> --breeze --stack=vue --database=mysql --pest --no-interaction
  cd <NAME>
  ```

  Extra Breeze/Inertia flags: `--dark`, `--ssr`, `--eslint`. For teams/2FA/API use `--jet` instead.

  > ⚠️ **Breeze targets the pre-13 skeleton — three fixes are needed for `npm run build` to pass on Laravel 13** (PHP/Inertia scaffolding installs fine; only the frontend toolchain needs reconciling):
  >
  > 1. **Do NOT pass `--typescript`** — the TS Breeze variant on L13 scaffolds an incomplete tree (only `app.ts`, no `bootstrap`, no `Pages/`) and the build fails. Use the plain JS stack.
  > 2. **Tailwind v3 → v4** (Breeze writes a TW3 setup over the skeleton's TW4):
  >    - `resources/css/app.css`: replace `@tailwind base/components/utilities;` with `@import 'tailwindcss';` (+ `@plugin '@tailwindcss/forms';` if forms is used).
  >    - `vite.config.js`: `import tailwindcss from '@tailwindcss/vite';` and add `tailwindcss()` to `plugins`.
  >    - Delete `tailwind.config.js` and `postcss.config.js`.
  >    - `package.json` devDependencies: keep `@tailwindcss/vite` **and** set `tailwindcss` to `^4` (it must stay installed — `@import 'tailwindcss'` resolves it); remove `autoprefixer` and `postcss`.
  > 3. **Remove `import './bootstrap';`** from `resources/js/app.js` — L13 ships no `resources/js/bootstrap.js`/axios, so the import is unresolved (Inertia doesn't need it).
  >
  > Then `npm install && npm run build` succeeds.

- **API / Microservice** (no starter kit):

  ```bash
  laravel new <NAME> --database=mysql --pest --no-interaction
  cd <NAME>
  ```

Confirm the exact flags for the installed version with `laravel new --help` (flag names vary between installer releases). Reliable flags here: `--breeze`/`--jet`, `--stack`, `--database=sqlite|mysql|mariadb|pgsql`, `--pest`/`--phpunit`, `--dark`/`--ssr`/`--typescript`/`--eslint`, `--git`, `--no-interaction`.

> Breeze scaffolds the full frontend (Inertia/TypeScript/Tailwind or Blade/Livewire), auth, Vite config and npm scripts. The manual stack files in `references/stacks/` are now **secondary** — use them only to customize/extend the generated structure, not to scaffold from scratch.
>
> **Already bundled in the default skeleton** (Laravel 13.x): `laravel/tinker`, `laravel/pail`, `laravel/pint`, and `laravel/pao`. Do not re-install these. `laravel/sail`, `laravel/breeze`, `laravel/jetstream` and `laravel/boost` are **not** bundled and are added on demand (Breeze/Jetstream via the `laravel new` flags above).

### 3. Configuration by type

#### For `--monolith`

The frontend is already scaffolded by **Breeze** via the `--breeze --stack=<...>` flags in step 2 (Inertia + Vue/React, or Blade/Livewire). No manual frontend bootstrapping is needed.

The stack files below are **secondary references** — use them only to extend or customize the structure Breeze produced (folder conventions, extra config, useful types):

- [references/stacks/blade.md](references/stacks/blade.md) — Blade/Livewire structure and conventions
- [references/stacks/vue-inertia.md](references/stacks/vue-inertia.md) — Vue + Inertia structure, composables, stores, useful types
- [references/stacks/react-inertia.md](references/stacks/react-inertia.md) — React + Inertia structure, hooks, contexts, useful types

Then build assets:

```bash
npm install && npm run build
```

#### For `--api`

```bash
php artisan install:api
```

This publishes `routes/api.php` + the Sanctum `personal_access_tokens` migration and adds `laravel/sanctum`.

> ⚠️ **`install:api` runs migrations at the end** (automatically with `--no-interaction`). This project uses **MySQL** (`--database=mysql`), so the database must be **up before this step**, otherwise it fails with a `QueryException` (the routes/migration/Sanctum publish is still done). **Set up Sail first, then migrate against it:**
> ```bash
> composer require laravel/sail --dev
> php artisan sail:install --with=mysql,redis --no-interaction
> ./vendor/bin/sail up -d
> ./vendor/bin/sail artisan install:api      # migrations now hit Sail's MySQL
> ```
> Ordering rule: with a MySQL project, do the **Docker/Sail step (4) before any migration** (`install:api`, `migrate`, `migrate:fresh`). Run artisan through `./vendor/bin/sail artisan …` so it targets the container DB.

**Cleanup**:
- Remove `resources/views/` (unless needed for emails)
- Remove everything frontend-related (vite, mix, node_modules, package.json)
- Remove sessions, cookies, CSRF from the middlewares
- `web.php` routes: keep only a health check route

**Configure**:
- Sanctum for API authentication
- API throttling
- Default JSON response format
- CORS

**Swagger / OpenAPI** (optional):
Ask: **"Do you want to install L5-Swagger for API documentation?"**

If yes:
```bash
composer require darkaonline/l5-swagger
php artisan vendor:publish --provider="L5Swagger\L5SwaggerServiceProvider"
```

**Sentry (error monitoring)** (optional):
Ask: **"Do you want to set up Sentry for monitoring?"**

If yes:
```bash
composer require sentry/sentry-laravel
php artisan sentry:publish --dsn=https://examplePublicKey@o0.ingest.sentry.io/0
```

#### For `--microservice`

Minimal installation:
- Remove everything frontend-related (views, vite, mix, node_modules, package.json)
- Remove sessions, cookies, CSRF
- Clean up the service providers (keep only the essentials)
- Health check: `/up` (Laravel 12 default)
- File cache locally, Redis in staging/production
- Sync queue locally, database or Redis in production

### 4. Common configuration (all types)

**Code quality**:
```bash
# PHPStan
composer require --dev phpstan/phpstan larastan/larastan
# Pint (formatter)
# (included by default with Laravel)
```

Create the configuration files:
- `phpstan.neon` with Larastan configuration
- `pint.json` with the project rules

**Tests**:
- The testing framework (Pest or PHPUnit) was chosen during `laravel new` — verify it is configured
- Create a basic test (health check)

**Docker (Laravel Sail — default)**:
Ask the user: **"Do you want to set up Docker via Laravel Sail?"**

If yes, use Sail (first-party Laravel Docker environment). **`laravel/sail` is not bundled in Laravel 13** — require it first, then run `sail:install` (without the package, `sail:install` errors with *"no commands defined in the sail namespace"*):

```bash
composer require laravel/sail --dev
php artisan sail:install --with=mysql,redis,mailpit --no-interaction
```

Common services: `mysql` / `mariadb` / `pgsql`, `redis`, `mailpit`, `meilisearch`, `minio`. Then start the stack:

```bash
./vendor/bin/sail up -d
```

This generates a ready-to-use **`compose.yaml`** (Laravel 13 uses `compose.yaml`, not `docker-compose.yml`). No custom Dockerfile/Nginx is required.

> 🔑 **Ordering rule (MySQL projects):** since the database lives in Sail, set up Sail and run `./vendor/bin/sail up -d` **before any migration** (`install:api`, `migrate`, `migrate:fresh --seed`). Run migrations via `./vendor/bin/sail artisan …` so they target the container's MySQL — running bare `php artisan migrate` on the host hits no DB and fails. Wait for the DB container to be `healthy` before migrating (Sail's mysql has a healthcheck; ~20–30s on first boot).
>
> 🔌 **Port conflicts:** Sail forwards `80` (app), `3306` (mysql), `6379` (redis), `8025` (mailpit) to the host by default. On a multi-project machine these are often already taken — set non-default forwarded ports in `.env` before `sail up`:
> ```env
> APP_PORT=8080
> FORWARD_DB_PORT=3307
> FORWARD_REDIS_PORT=6380
> ```

**Custom Docker (advanced — optional, for K8s production)**:
Only if the project needs a production-grade K8s deployment with the in-house stack (PHP-FPM + Nginx/Apache, Xdebug, Helm), ask: **"Do you also want the custom production Docker setup (K8s)?"**

If yes, follow [references/examples/docker.md](references/examples/docker.md):
- `Dockerfile` / `Dockerfile-http` (production), `.dockerignore`
- `.devops/docker/` configs (PHP, Xdebug, Nginx) and DB init scripts
- This pairs with the `.devops` Helm values and the custom-script Makefile variant.

**CI/CD (GitHub Actions — default)**:
Ask the user: **"Do you want to set up CI (GitHub Actions)?"**

The marketplace is multi-forge, but projects are most likely hosted on GitHub, so **do not create `.gitlab-ci.yml` by default**.

If yes, follow [references/examples/ci.md](references/examples/ci.md):
- Create `.github/workflows/ci.yml` (Pint + PHPStan + tests)
- Create `.github/pull_request_template.md`
- **GitLab** (`.gitlab-ci.yml` + merge request template) only if the user explicitly hosts on GitLab — the example is in the same reference file.

**.devops (infrastructure — optional)**:
Ask the user: **"Do you want to set up the .devops folder (git hooks, helm)?"**

If yes, follow the [references/examples/devops.md](references/examples/devops.md) guide:
- Create the git hooks in `.devops/git/hooks/` (pre-commit with Pint, semantic commit-msg, post-merge)
- Create the Helm values in `.devops/helm/` (values.yaml, values.review.yaml, values.staging.yaml, values.production.yaml) — only with the custom Docker / K8s option
- The `.devops/docker/scripts/` helpers are only needed for the **custom Docker** setup; with Sail, use `sail` directly instead.

**Makefile**:
Ask the user: **"Do you want to set up a Makefile?"**

If yes, follow the [references/examples/makefile.md](references/examples/makefile.md) guide. Pick the variant that matches the Docker choice:
- **Sail (default)**: targets wrap `./vendor/bin/sail` (up/down, artisan, composer, npm, test, pint, phpstan).
- **Custom Docker (K8s)**: the in-house variant using `.devops/docker/scripts/` (monolith: with NPM; API/microservice: with hadolint and dotenv-linter, no NPM).

### 5. AI tooling (optional)

Ask the user: **"Do you want to set up Laravel's AI tooling?"** — these are independent, offer each:

**Laravel Boost** (recommended for agent-assisted dev) — gives AI agents Laravel-specific context, tools, and version-aware docs:

```bash
composer require laravel/boost --dev
php artisan boost:install --no-interaction
```

`boost:install --no-interaction` runs **fully non-interactively**: it auto-detects the IDE/agents (Claude Code included), installs the relevant AI guidelines + agent skills + MCP server config, and writes `CLAUDE.md`, `.mcp.json` and `.claude/`. Scope it with `--guidelines` / `--skills` / `--mcp` if you only want some of these. Custom guidelines go in `.ai/guidelines/*`.

> Because Boost generates `CLAUDE.md` itself, run it **before** step 6 (AI files) and only fill the gaps it leaves — do not overwrite Boost's output.

**Laravel PAO** (token-efficient tool output for agents) — compresses test/PHPStan/Rector output to compact JSON when run inside an AI agent. **Already bundled by default in Laravel 13.x** — just verify it is present (`composer show laravel/pao`); only install it on older projects that lack it:

```bash
composer require laravel/pao --dev   # only if missing
```

Zero-config: auto-activates only in detected AI agent environments.

**Laravel AI SDK** (only if the application itself consumes AI) — first-party toolkit for text generation, agents, structured output, embeddings/RAG:

```bash
composer require laravel/ai
```

> Boost and PAO are about *developing* with AI agents; the AI SDK is for *building* AI features into the app. Don't install the AI SDK unless the project needs AI capabilities.

### 6. AI files configuration

> If Laravel Boost was installed (step 5), `boost:install` may already have generated/updated `CLAUDE.md` and agent guidelines — review them before overwriting and only fill the gaps below.

**`CLAUDE.md`** at the root:
```markdown
# Project Configuration

## Repository

- **Host**: GitHub (default) — set the remote URL once created
- **Main Branch**: main

## Description

<project description>

## Tech Stack

- PHP <version>
- Laravel <version>
- <Frontend starter kit if monolith>
- <Database>
- Local env: Laravel Sail
```

**`AGENTS.md`** at the root:
```markdown
# <Project name>

<Short description>

## Build & Test

- Start: `./vendor/bin/sail up -d`
- Install: `composer install && npm install`
- Test: `./vendor/bin/sail artisan test`
- Lint: `./vendor/bin/pint --test`
- Static analysis: `./vendor/bin/phpstan analyse`

## Code Conventions

- PSR-12
- Strict types (`declare(strict_types=1)`)
- Laravel conventions
```

**Project rules** (`.claude/rules/` or `.agents/rules/`):
- Offer to run `/setup-rules` to generate the conventions

### 7. Git initialization

If `laravel new` was run without `--git`, initialize the repository:

```bash
git init
git add -A
git commit -m "chore: initialisation du projet Laravel"
```

### 8. Final summary

```
Laravel project "<NAME>" initialized ✅

  Type: Monolith (Vue.js + Inertia)
  Database: MySQL
  Frontend: Vue 3 + Inertia.js + Tailwind CSS (starter kit)
  Local env: Laravel Sail

  Structure:
  ├── app/          (Laravel backend)
  ├── resources/js/ (Vue.js frontend)
  ├── tests/        (Pest / PHPUnit)
  ├── CLAUDE.md     (AI config)
  ├── AGENTS.md     (agent conventions)
  └── compose.yaml (Sail)

  Next steps:
  1. Start the stack: ./vendor/bin/sail up -d
  2. Run /setup-rules to generate the conventions
  3. Create the GitHub repository and push
```

## Usage Examples

```bash
# Interactive installation (asks everything)
/init-laravel

# Vue.js monolith
/init-laravel --monolith --name=mon-app

# REST API
/init-laravel --api --name=mon-api

# Microservice
/init-laravel --microservice --name=ms-users
```
