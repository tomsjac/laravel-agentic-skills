# Makefile - Example files

Example Makefiles for Laravel projects.

- **Sail (default)**: thin wrappers around `./vendor/bin/sail` — use this when Docker is provided by Laravel Sail.
- **Custom Docker (K8s)**: the in-house variants using `.devops/docker/scripts/` — use these only with the custom production Docker setup. Two flavors: monolith (with frontend) and microservice/API (without frontend).

## Makefile - Sail (default)

```makefile
# -- Include .env
ifneq ("$(wildcard .env)","")
	include .env
endif

# Executables
SAIL          = ./vendor/bin/sail
ARTISAN       = $(SAIL) artisan
COMPOSER      = $(SAIL) composer
NPM           = $(SAIL) npm
GIT           = git

# Misc
.DEFAULT_GOAL = help
.PHONY        :

##! —— Helpers ———————————————————————————————————
help: ##! Outputs this help screen
	@awk 'BEGIN {FS = ":.*?##!"; printf "\nUsage:\n  make \033[36m\033[0m\n"} /(^[a-zA-Z0-9@_-]+:.*?##!.*$$)/  { printf "\033[32m%-30s\033[0m %s\n", $$1, $$2 } /(^##!)/ { printf "\n\033[33m%s\033[0m\n", substr($$1,4)} ' Makefile

##! —— Sail / Docker —————————————————————————————————————————————————————————
up: ##! Start the Sail stack
	$(SAIL) up -d

down: ##! Stop the Sail stack
	$(SAIL) down

build: ##! Rebuild the Sail images
	$(SAIL) build --no-cache

logs: ##! Show live logs
	$(SAIL) logs -f

shell: ##! Open a shell in the app container
	$(SAIL) shell

##! —— Project ———————————————————————————————————————————————————————————————
install: ##! First-time install
	cp -n .env.example .env || true
	$(SAIL) up -d
	$(COMPOSER) install
	$(ARTISAN) key:generate
	$(NPM) install
	$(NPM) run build
	$(ARTISAN) migrate --seed

update: ##! Update dependencies and rebuild assets
	$(COMPOSER) install
	$(NPM) install
	$(NPM) run build
	$(ARTISAN) optimize:clear

fresh: ##! Reset the database with seeders
	$(ARTISAN) migrate:fresh --seed

dev: ##! Start the Vite dev server
	$(NPM) run dev

tinker: ##! Launch Tinker
	$(ARTISAN) tinker

##! —— Tests & Quality ———————————————————————————————————————————————————————
test: ##! Run the test suite
	$(ARTISAN) test

stan: ##! Run PHPStan
	$(SAIL) php ./vendor/bin/phpstan analyse --memory-limit 1G

cs: ##! Lint PHP with Pint
	$(SAIL) php ./vendor/bin/pint --test

cs-fix: ##! Fix PHP with Pint
	$(SAIL) php ./vendor/bin/pint

check: cs stan test ##! Run all checks before committing
```

> **API / Microservice with Sail**: drop the `NPM`, `dev`, and `npm`-related targets.

## Makefile - Monolith (custom Docker, with frontend)

```makefile
# -- Include .env for URL APP
ifneq ("$(wildcard .env)","")
	include .env
else
	include .env.local
endif

# —— Inspired by ———————————————————————————————————————————————————————————————
# https://www.strangebuzz.com/fr/snippets/le-makefile-parfait-pour-symfony
# Setup ————————————————————————————————————————————————————————————————————————

# MakeFile
MAKEFILE_LIST = Makefile

# Executables
EXEC_PHP      = .devops/docker/scripts/php
COMPOSER      = .devops/docker/scripts/composer
NPM           = .devops/docker/scripts/npm
NPX           = .devops/docker/scripts/npx
YARN          = .devops/docker/scripts/yarn
NODE          = .devops/docker/scripts/node
GIT           = git

# Alias
ARTISAN       = $(EXEC_PHP) artisan

# Executables: vendors
PHPUNIT       = @$(EXEC_PHP) ./vendor/bin/phpunit
PHPSTAN       = @$(EXEC_PHP) ./vendor/bin/phpstan
PINT          = @$(EXEC_PHP) ./vendor/bin/pint

# Executables: local only
DOCKER        = docker
DOCKER_COMP   = docker-compose

# Misc
.DEFAULT_GOAL = help
.PHONY        :

##! —— Helpers ———————————————————————————————————
help: ##! Outputs this help screen
	@awk 'BEGIN {FS = ":.*?##!"; printf "\nUsage:\n  make \033[36m\033[0m\n"} /(^[a-zA-Z0-9@_-]+:.*?##!.*$$)/  { printf "\033[32m%-30s\033[0m %s\n", $$1, $$2 } /(^##!)/ { printf "\n\033[33m%s\033[0m\n", substr($$1,4)} ' $(MAKEFILE_LIST)

##! —— Docker ————————————————————————————————————————————————————————————————
docker@network: ##! Start network
	$(DOCKER) network inspect proxy > /dev/null || $(DOCKER) network create proxy
	$(DOCKER) network inspect app-network > /dev/null || $(DOCKER) network create app-network

docker@up: ##! Start the docker hub (PHP, MySQL, redis ...)
	make docker@network
	make docker@down
	$(DOCKER_COMP) up -d --remove-orphans

docker@build: ##! Builds the images
	$(DOCKER_COMP) build --pull --no-cache

docker@down: ##! Stop the docker hub
	$(DOCKER_COMP) down --remove-orphans

docker@logs: ##! Show live logs
	@$(DOCKER_COMP) logs --tail=0 --follow

docker@clean: ##! Clean volumes docker
	@$(DOCKER_COMP) down --volumes --remove-orphans --rmi local

docker@deleteDatabase: ##! Delete the project database
	make docker@down
	@$(DOCKER) volume rm {{COMPOSE_PROJECT_NAME}}_mysql-data

##! —— Composer ————————————————————————————————————————————————————————————
composer@install: composer.lock ##! Install vendors according to the current composer.lock file
	@$(COMPOSER) install --no-progress --prefer-dist --optimize-autoloader

##! —— NPM / JavaScript —————————————————————————————————————————————————————
npm@install: package-lock.json ##! Install packages according to the current package-lock.json
	@$(NPM) install --check-files

npm@dev: ##! Rebuild assets for the dev env
	make npm@install
	@$(NPM) run dev

npm@watch: ##! Starts the live asset build command
	@$(NPM) run dev

npm@check: ##! Check for package updates and usage
	@$(NPM) run check

##! —— Laravel  ———————————————————————————————————————————————————————————————
laravel@cac: ##! Clear All Cache
	$(ARTISAN) config:clear
	$(ARTISAN) route:clear
	$(ARTISAN) event:clear
	$(ARTISAN) clear-compiled

laravel@cache: ##! Create Cache
	$(ARTISAN) config:cache
	$(ARTISAN) route:cache
	$(ARTISAN) event:cache
	$(ARTISAN) package:discover

laravel@horizon: ##! Launch Horizon
	@$(ARTISAN) 'horizon:terminate';
	@$(ARTISAN) 'cache:clear';
	@$(ARTISAN) 'horizon';

laravel@helper: ##! Generate IDE Helper
	@$(ARTISAN) 'ide-helper:generate'
	@$(ARTISAN) 'ide-helper:meta'

laravel@tinker: ##! Launch Tinker Console
	@$(ARTISAN) 'tinker'

##! —— Project  ———————————————————————————————————————————————————————————————
app@start: ##! Start project
	make docker@up ## Start Docker
	@powershell.exe /c start $(APP_URL)

app@install: ##! Installation of the project if it is the first time
	make app@envLocal ##Add local env
	make docker@up ## Start Docker
	##### Install Packages
	make app@update
	##### Init
	@$(ARTISAN) storage:link
	make git@hooks-install ## Hook install
	@$(ARTISAN) key:generate --ansi
	##### Database
	@$(ARTISAN) migrate:fresh --seed
	##### Finish
	@$(ARTISAN) env ## Congratulation
	@powershell.exe /c start $(APP_URL)

app@update: ##! Update project
	make composer@install ## Composer Update
	make npm@dev ## Generate Assets
	make laravel@cac ## Clear cache

app@reset: ##! Reset project
	make app@envLocal ##Add local env
	make app@update
	@$(ARTISAN) migrate:fresh --seed

app@envLocal: ##! Installation of the local env
	-rm -f .env
	cp .env.local .env

app@check: ##! Check the code before committing
	make code@stan
	make test@run

##! —— Tests —————————————————————————————————————————————————————————————————
test@run: phpunit.xml ##! Run All tests
	@$(ARTISAN) test

test@unit: phpunit.xml ##! Run Unit tests
	@$(PHPUNIT) --testsuite unit --stop-on-failure --colors=always

test@feature: phpunit.xml ##! Run Feature tests
	@$(PHPUNIT) --testsuite feature --stop-on-failure --colors=always

##! —— Coding standards ——————————————————————————————————————————————————————
code@cs: code@php-formatter-lint code@js-formatter-lint ##! Run all coding standards checks

code@stan: ##! Run PHPStan
	@$(PHPSTAN) analyse -c phpstan.neon --memory-limit 1G

code@php-formatter-lint: ##! Lint PHP files with Laravel Pint
	@$(PINT) --test

code@php-formatter-fix: ##! Fix PHP files with Laravel Pint
	@$(PINT)

code@js-formatter-lint: ##! Lint js/vue files with eslint
	@$(NPM) run lint

code@js-formatter-fix: ##! Fix js/vue files with eslint
	@$(NPM) run fix

##! —— Deploy & Prod —————————————————————————————————————————————————————————
deploy@review: ##! Deployment in review (K8s)
	@echo "EXEC PHP : $(EXEC_PHP)"
	make deploy@staging

deploy@staging: ##! Deployment in staging (K8s)
	@echo "EXEC PHP : $(EXEC_PHP)"
	$(ARTISAN) migrate --force
	@$(ARTISAN) operations:process --sync

deploy@production: ##! Deployment in production
	@echo "EXEC PHP : $(EXEC_PHP)"
	make laravel@cac
	$(ARTISAN) migrate --force
	@$(ARTISAN) operations:process --sync
	make laravel@cache

##! —— GIT  —————————————————————————————————————————————————————————————————
git@stats: ##! Commits by the hour for the main author of this project
	@$(GIT) log --author="$(GIT_AUTHOR)" --date=iso | perl -nalE 'if (/^Date:\s+[\d-]{10}\s(\d{2})/) { say $$1+0 }' | sort | uniq -c|perl -MList::Util=max -nalE '$$h{$$F[1]} = $$F[0]; }{ $$m = max values %h; foreach (0..23) { $$h{$$_} = 0 if not exists $$h{$$_} } foreach (sort {$$a <=> $$b } keys %h) { say sprintf "%02d - %4d %s", $$_, $$h{$$_}, "*"x ($$h{$$_} / $$m * 50); }'

git@hooks-install: ##! Installation of GIT hooks
	.devops/git/hooks/install.sh
```

## Makefile - Microservice / API (custom Docker, without frontend)

```makefile
# -- Include .env for URL APP
ifneq ("$(wildcard .env)","")
	include .env
else
	include .env.local
endif

# —— Inspired by ———————————————————————————————————————————————————————————————
# https://www.strangebuzz.com/fr/snippets/le-makefile-parfait-pour-symfony
# Setup ————————————————————————————————————————————————————————————————————————

# MakeFile
MAKEFILE_LIST = Makefile

# Executables
EXEC_PHP      = .devops/docker/scripts/php
COMPOSER      = .devops/docker/scripts/composer
GIT           = git
HADOLINT      = .devops/docker/scripts/hadolint
DOTENV_LINTER = .devops/docker/scripts/dotenv-linter

# Alias
ARTISAN       = $(EXEC_PHP) artisan

# Executables: vendors
PHPUNIT       = @$(EXEC_PHP) ./vendor/bin/phpunit
PHPSTAN       = @$(EXEC_PHP) ./vendor/bin/phpstan
LARAVEL_PINT  = @$(EXEC_PHP) ./vendor/bin/pint

# Executables: local only
DOCKER        = docker
DOCKER_COMP   = docker-compose

# Misc
.DEFAULT_GOAL = help
.PHONY        :

##! —— Helpers ———————————————————————————————————
help: ##! Outputs this help screen
	@awk 'BEGIN {FS = ":.*?##!"; printf "\nUsage:\n  make \033[36m\033[0m\n"} /(^[a-zA-Z0-9@_-]+:.*?##!.*$$)/  { printf "\033[32m%-30s\033[0m %s\n", $$1, $$2 } /(^##!)/ { printf "\n\033[33m%s\033[0m\n", substr($$1,4)} ' $(MAKEFILE_LIST)

##! —— Docker ————————————————————————————————————————————————————————————————
docker@network: ##! Start network
	$(DOCKER) network inspect proxy > /dev/null || $(DOCKER) network create proxy
	$(DOCKER) network inspect app-network > /dev/null || $(DOCKER) network create app-network

docker@up: ##! Start the docker hub (PHP, MySQL, redis ...)
	make docker@network
	$(DOCKER_COMP) up -d --remove-orphans --force-recreate

docker@build: ##! Builds the images
	$(DOCKER_COMP) build --pull --no-cache

docker@down: ##! Stop the docker hub
	$(DOCKER_COMP) down --remove-orphans

docker@logs: ##! Show live logs
	@$(DOCKER_COMP) logs --tail=0 --follow

docker@clean: ##! Clean volumes docker
	@$(DOCKER_COMP) down --volumes --remove-orphans --rmi local

docker@deleteDatabase: ##! Delete the project database
	make docker@down
	@$(DOCKER) volume rm ${DB_DATABASE}_mysql-data

##! —— Composer ————————————————————————————————————————————————————————————
composer@install: composer.lock ##! Install vendors according to the current composer.lock file
	@$(COMPOSER) install --no-progress --prefer-dist --optimize-autoloader

##! —— Laravel  ———————————————————————————————————————————————————————————————
laravel@cac: ##! Clear All Cache
	@$(ARTISAN) 'config:clear';
	@$(ARTISAN) 'view:clear';
	@$(ARTISAN) 'route:clear';

laravel@horizon: ##! Launch Horizon
	@$(ARTISAN) 'horizon:terminate';
	@$(ARTISAN) 'cache:clear';
	@$(ARTISAN) 'redis:flush';
	@$(ARTISAN) 'horizon';

laravel@helper: ##! Generate IDE Helper
	@$(ARTISAN) 'ide-helper:generate'
	@$(ARTISAN) 'ide-helper:meta'

laravel@tinker: ##! Launch Tinker Console
	@$(ARTISAN) 'tinker'

##! —— Project  ———————————————————————————————————————————————————————————————
app@start: ##! Start project
	make docker@up ## Start Docker
	@powershell.exe /c start $(APP_URL)

app@install: ##! Installation of the project if it is the first time
	make app@envLocal ##Add local env
	make docker@up ## Start Docker
	##### Install Packages
	make app@update
	##### Init
	@$(ARTISAN) storage:link
	make git@hooks-install ## Hook install
	@$(ARTISAN) key:generate --ansi
	##### Database
	make app@resetDb
	##### Finish
	@$(ARTISAN) env ## Congratulation
	@powershell.exe /c start $(APP_URL)

app@update: ##! Update project
	make composer@install ## Composer Update
	make laravel@cac ## Clear cache

app@reset: ##! Reset project
	make app@envLocal ##Add local env
	make app@update
	make app@resetDb

app@envLocal: ##! Installation of the local env
	-rm -f .env
	cp .env.local .env

app@resetDb: ##! Reset Database with seeders
	@$(ARTISAN) migrate:fresh --seed

app@check: ##! Check the code before committing
	@$(DOTENV_LINTER) fix --no-backup --skip UnorderedKey
	@$(HADOLINT) < Dockerfile
	@$(HADOLINT) < Dockerfile-http
	make code@stan
	make test@run

##! —— Tests —————————————————————————————————————————————————————————————————
test@run: phpunit.xml ##! Run All tests
	@$(ARTISAN) test

test@unit: phpunit.xml ##! Run Unit tests
	@$(PHPUNIT) --testsuite unit --stop-on-failure --colors=always

test@feature: phpunit.xml ##! Run Feature tests
	@$(PHPUNIT) --testsuite feature --stop-on-failure --colors=always

##! —— Coding standards ——————————————————————————————————————————————————————
code@cs: code@php-formatter-lint ##! Run all coding standards checks

code@stan: ##! Run PHPStan
	@$(PHPSTAN) analyse -c phpstan.neon --memory-limit 1G

code@php-formatter-lint: ##! Lint PHP files with laravel pint
	@$(LARAVEL_PINT) --test

code@php-formatter-fix: ##! Fix PHP files with laravel pint
	@$(LARAVEL_PINT)

##! —— Deploy & Prod —————————————————————————————————————————————————————————
deploy@review: ##! Deployment in review
	$(ARTISAN) migrate --force
	@$(ARTISAN) operations:process --sync

deploy@staging: ##! Deployment in pre-prod
	@echo "EXEC PHP : $(EXEC_PHP)"
	$(ARTISAN) migrate --force
	@$(ARTISAN) operations:process --sync

deploy@production: ##! Deployment in production
	@echo "EXEC PHP : $(EXEC_PHP)"
	$(ARTISAN) migrate --force
	@$(ARTISAN) operations:process --sync

##! —— GIT  —————————————————————————————————————————————————————————————————
git@stats: ##! Commits by the hour for the main author of this project
	@$(GIT) log --author="$(GIT_AUTHOR)" --date=iso | perl -nalE 'if (/^Date:\s+[\d-]{10}\s(\d{2})/) { say $$1+0 }' | sort | uniq -c|perl -MList::Util=max -nalE '$$h{$$F[1]} = $$F[0]; }{ $$m = max values %h; foreach (0..23) { $$h{$$_} = 0 if not exists $$h{$$_} } foreach (sort {$$a <=> $$b } keys %h) { say sprintf "%02d - %4d %s", $$_, $$h{$$_}, "*"x ($$h{$$_} / $$m * 50); }'

git@hooks-install: ##! Installation of GIT hooks
	.devops/git/hooks/install.sh
```
