# .devops - Example files

The `.devops/` folder centralizes all local infrastructure and deployment configuration. Adapt the placeholders to the project.

## .devops folder structure

```
.devops/
├── docker/
│   ├── Dockerfile              # Dockerfile de développement (avec xdebug)
│   ├── php/
│   │   └── custom.ini          # Configuration PHP custom
│   ├── xdebug/
│   │   └── xdebug.ini          # Configuration Xdebug
│   ├── nginx/
│   │   └── default             # Config Nginx (monolithe uniquement)
│   ├── mysql-init/             # Scripts init DB (monolithe)
│   │   ├── create_dev.sh
│   │   └── z-flush.sql
│   ├── mysql-initdb/           # Scripts init DB (microservice/API)
│   │   ├── 001-init.sh
│   │   └── 999-flush.sql
│   └── scripts/
│       ├── php                 # Raccourci : docker compose exec php-fpm php
│       ├── composer            # Raccourci : docker compose exec php-fpm composer
│       ├── npm                 # Raccourci : docker run node npm (monolithe)
│       ├── npx                 # Raccourci : docker run node npx (monolithe)
│       ├── node                # Raccourci : docker run node (monolithe)
│       ├── _nodeImage          # Helper : node container sans port
│       └── _nodeImageVite      # Helper : node container avec port Vite 5173
├── git/
│   └── hooks/
│       ├── install.sh          # Installation des hooks Git
│       ├── pre-commit          # Hook pre-commit (Pint)
│       ├── commit-msg          # Hook commit-msg (semantic commits)
│       ├── post-merge          # Hook post-merge (composer install)
│       ├── _branchs            # Helper : détection type de branche
│       └── _helpers            # Helper : affichage formaté
├── helm/
│   ├── values.yaml             # Config K8s de base
│   ├── values.review.yaml      # Config K8s review
│   ├── values.staging.yaml     # Config K8s staging
│   └── values.production.yaml  # Config K8s production
├── php/
│   └── custom.ini              # Config PHP (microservice simplifié)
└── tinker/
    └── psysh/
        └── psysh_history       # Historique Tinker
```

## Scripts Docker

### .devops/docker/scripts/php

```bash
#!/bin/bash
source .env;docker compose exec php-fpm php $@
```

### .devops/docker/scripts/composer

```bash
#!/bin/bash
source .env;docker compose exec php-fpm php /usr/local/bin/composer $@
```

### .devops/docker/scripts/npm (monolithe uniquement)

```bash
#!/bin/bash
. .devops/docker/scripts/_nodeImageVite npm $@
```

### .devops/docker/scripts/npx (monolithe uniquement)

```bash
#!/bin/bash
. .devops/docker/scripts/_nodeImage npx $@
```

### .devops/docker/scripts/node (monolithe uniquement)

```bash
#!/bin/bash
. .devops/docker/scripts/_nodeImage node $@
```

### .devops/docker/scripts/_nodeImage

```bash
#!/bin/bash
docker run --rm -ti -w /var/www -v $(pwd):/var/www node:20 $@
```

### .devops/docker/scripts/_nodeImageVite

```bash
#!/bin/bash
docker run -p 5173:5173 --rm -ti -w /var/www -v $(pwd):/var/www node:20 $@
```

### .devops/docker/scripts/hadolint

```bash
#!/bin/bash
## See https://hub.docker.com/r/hadolint/hadolint - How to use
docker run --rm -i hadolint/hadolint:2.12.0 $@
```

### .devops/docker/scripts/dotenv-linter

```bash
#!/bin/bash
docker run --rm -ti -w /app -v $(pwd):/app dotenvlinter/dotenv-linter:3.3.0 $@
```

## Git Hooks

### .devops/git/hooks/install.sh

```bash
#!/bin/bash
. .devops/git/hooks/_helpers

titleDisplay "Install GIT hooks"

#############################################
# ########### Pre commit
cat << EOF > .git/hooks/pre-commit
#!/bin/bash
. .devops/git/hooks/pre-commit
EOF
chmod u+x .devops/git/hooks/pre-commit
chmod u+x .git/hooks/pre-commit

#############################################
# ########### Commit MSG
cat << EOF > .git/hooks/commit-msg
#!/bin/bash
. .devops/git/hooks/commit-msg
EOF

chmod u+x .devops/git/hooks/commit-msg
chmod u+x .git/hooks/commit-msg

#############################################
# ########### Post merge
cat << EOF > .git/hooks/post-merge
#!/bin/bash
. .devops/git/hooks/post-merge
EOF

titleDisplay "it's ok"
```

### .devops/git/hooks/_helpers

```bash
#!/bin/bash

titleDisplay(){
    printf "\033[36m ### $1 \033[0m \n";
}

sectionDisplay(){
    printf '\033[0;32m'
    printf "=%.0s"  $(seq 1 64)
    printf "\n %-50s : %10s\n" "$1"  "$2"
    printf "=%.0s"  $(seq 1 64)
    printf '\033[0m \n'
}
```

### .devops/git/hooks/_branchs

```bash
#!/bin/bash
local_branch=$(git name-rev --name-only HEAD)

feature_regex="^([0-9.x/]+?)(feature|bugfix|improvement|hotfix)-([a-z0-9.\_\-\#]+)"
main_regex="^(main|master)(.*)"
demo_regex="^(demo)(.*)"

fn_onlyFeature (){
    if [[ ! "$local_branch" =~ ${feature_regex} ]]; then
        exit 0
    fi
}

fn_onlyDemo (){
    if [[ ! "$local_branch" =~ ${demo_regex} ]]; then
        exit 0
    fi
}

fn_onlyMain (){
    if [[ ! "$local_branch" =~ ${main_regex} ]]; then
        exit 0
    fi
}
```

### .devops/git/hooks/pre-commit

```bash
#!/bin/bash

# only feature branch
. .devops/git/hooks/_branchs
. .devops/git/hooks/_helpers
fn_onlyFeature

# Allows us to read user input below, assigns stdin to keyboard
exec < /dev/tty

# PHP CS FIXER
runPhpCsFixerOn () {
    titleDisplay 'PHP fixer with Laravel Pint'

    LARAVEL_PINT=".devops/docker/scripts/php ./vendor/bin/pint"

    $LARAVEL_PINT;
}

is_files=$(git diff --cached --name-only)
if [ -n "$is_files" ]; then

    sectionDisplay 'Pre-commit verification ...', 'Start'

    changed_files=$(git diff --staged --name-only --diff-filter=ACM -- '*.php')

    if [ -n "$changed_files" ]; then
        runPhpCsFixerOn $changed_files
        git add $changed_files
    fi

    sectionDisplay ' Pre-commit verification', 'End'
fi

exec <&-
```

### .devops/git/hooks/commit-msg

```bash
#!/bin/bash

# only feature branch
. .devops/git/hooks/_branchs
fn_onlyFeature

# Color codes
red='\033[0;31m'
yellow='\033[0;33m'
blue='\033[0;34m'
green='\033[0;32m'
NC='\033[0m' # No colors

commit_regex='^((chore|docs|feat|fix|merge|perf|refact|refactor|style|test|revert|ci|build)(\([a-zA-Z0-9]+\))?:|merge)'

if ! grep -iqE "$commit_regex" "$1"; then
  printf "${red}Please use semantic commit messages:\n"
  printf "${yellow}feat${NC}[${green}(scope)${NC}]: ${blue}add hat wobble\n"
  printf "${yellow}^--^${green} ^--*--^ ${blue}  ^------------^ -> Summary in present tense.\n"
  printf "${yellow} *      ${green}*-> [optional]: Scope of the commit.\n"
  printf "${yellow} *-> Type: feat, fix, build, ci, docs, perf, refactor, style, test, or chore.\n${NC}"
  exit 1
fi

exit 0
```

### .devops/git/hooks/post-merge

```bash
#!/bin/bash
. .devops/git/hooks/_helpers

changedFiles="$(git diff-tree -r --name-only --no-commit-id ORIG_HEAD HEAD)"
runOnChange() {
	echo "$changedFiles" | grep -q "$1"
}

# only feature branch
. .devops/git/hooks/_branchs
fn_onlyFeature

# Allows us to read user input below, assigns stdin to keyboard
exec < /dev/tty

if runOnChange 'composer.lock'; then
     titleDisplay 'Composer update'
    .devops/docker/scripts/composer install
fi

exec <&-
```

## Helm Values

### .devops/helm/values.yaml

```yaml
# Authentification HTTP
httpAuth:
  enabled: false
  user: app
  password: paris

# Configuration du cron Laravel
laravelCron:
  enabled: true

# Configuration de Horizon Laravel
laravelHorizon:
  enabled: true

# Configuration de Laravel Reverb
laravelReverb:
  enabled: false

# Configuration de MariaDB
mariadb:
  enabled: true
  primary:
    persistence:
      size: 1Gi

# Configuration de Redis
redis:
  enabled: true
  master:
    persistence:
      size: 1Gi

# Configuration d'Elasticsearch
elasticsearch:
  enabled: false
  master:
    persistence:
      size: 1Gi
  data:
    persistence:
      size: 1Gi

# Configuration du serveur web
webserver:
  enabled: true

# Configuration de Maildev
maildev:
  enabled: false
  maildev:
    config:
      web:
        disabled: false
        password: plusquepro
        username: app
      smtp:
        incoming:
          password: password4smtp

# Configuration de MinIO
minio:
  enabled: false
  persistence:
    size: 1Gi

# Configuration de phpMyAdmin
phpmyadmin:
  enabled: false

# Configuration de MeiliSearch
meilisearch:
  enabled: false
  persistence:
    size: 1Gi
```

> **Note**: Enable/disable the services based on the project's needs. For a simple microservice, disable Horizon, MeiliSearch, MinIO, etc.

### .devops/helm/values.review.yaml

```yaml
upgradeCommands:
  - /bin/sh
  - -c
  - make EXEC_PHP=php deploy@review

phpmyadmin:
  enabled: true
```

### .devops/helm/values.staging.yaml

```yaml
upgradeCommands:
  - /bin/sh
  - -c
  - make EXEC_PHP=php deploy@staging

phpmyadmin:
  enabled: true
```

### .devops/helm/values.production.yaml

```yaml
upgradeCommands:
  - /bin/sh
  - -c
  - make EXEC_PHP=php deploy@production
```

## PHP Config (simplified microservice)

### .devops/php/custom.ini

```ini
opcache.enable=0
```

> For microservices without the `docker/` folder, place `custom.ini` directly in `.devops/php/`.
