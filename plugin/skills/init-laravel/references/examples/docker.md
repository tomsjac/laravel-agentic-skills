# Docker - Custom setup (advanced / K8s production)

> **Default is Laravel Sail.** For local development, prefer `composer require laravel/sail --dev` then `php artisan sail:install --with=mysql,redis,mailpit` and `./vendor/bin/sail up -d` — it generates a ready-to-use `compose.yaml` with no custom files (`laravel/sail` is not bundled in Laravel 13, so require it first). See the [Sail docs](https://laravel.com/docs/sail).
>
> Use the configuration below **only** when the project needs the in-house production-grade stack (PHP-FPM + Nginx/Apache, Xdebug, K8s/Helm deployment) that Sail does not cover.

Example Docker configurations for Laravel projects. Adapt the `{{PROJECT_NAME}}`, `{{PHP_VERSION}}`, `{{DB_VERSION}}` placeholders to the project.

## Dockerfile (Production - PHP-FPM)

```dockerfile
FROM php:{{PHP_VERSION}}-fpm
ARG TAG
ENV APP_TAG $TAG

COPY --chown=33:33 . /var/www
RUN if [ ! -d storage/framework/sessions ];then mkdir -p storage/framework/sessions;chmod g+w -R storage/framework/sessions;fi; \
    if [ ! -d storage/framework/views ];then mkdir -p storage/framework/views;chmod g+w -R storage/framework/views;fi; \
    if [ ! -d storage/framework/cache ];then mkdir -p storage/framework/cache;chmod g+w -R storage/framework/cache;fi; \
    if [ ! -d bootstrap/cache ];then mkdir -p bootstrap/cache;chmod g+w -R bootstrap/cache;fi; \
    if [ ! -d storage/logs ];then mkdir -p storage/logs;chmod g+w -R storage/logs;fi
RUN sed -i 's/;user/user/g' /usr/local/etc/php-fpm.d/www.conf; \
    sed -i 's/;group/group/g' /usr/local/etc/php-fpm.d/www.conf; \
    sed -i 's/pm.max_children = 5/pm.max_children = 30/g' /usr/local/etc/php-fpm.d/www.conf

# Commande Global PHP-FPM
RUN php artisan storage:link; \
    php artisan env

ENV DOCUMENT_ROOT=/var/www/public
```

> **Monolith note**: For a monolith with a frontend (Blade/Vue/React), add caching for routes, views, and events:
> ```dockerfile
> RUN php artisan storage:link; \
>     php artisan route:cache; \
>     php artisan view:cache; \
>     php artisan event:cache;
> ```

## Dockerfile-http (Apache - static assets)

```dockerfile
FROM apache:2.4-php-index
COPY public/ /var/www/public/
```

## .dockerignore

```
# Environment files - exclude all except main .env
.env.*
!.env

# Logs
logs/
*.log

# Runtime data
pids/
*.pid
*.seed

# Coverage directory used by tools like istanbul
coverage/

# Development files
.git/
.gitignore
README.md
*.md

# IDE files
.vscode/
.idea/
*.swp
*.swo

# Build artifacts
storage/logs/
storage/app/
storage/framework/cache/
storage/framework/sessions/
storage/framework/testing/
storage/framework/views/
bootstrap/cache/

# Docker
docker-compose.yml
.devops/
Dockerfile*

# Temporary files
tmp/
temp/
```

## docker-compose.yml - Monolith (with Nginx, frontend)

```yaml
services:

  ### PHP-FPM : For IDE (phpstorm) ####################################
  php-fpm-ide:
    container_name: ${COMPOSE_PROJECT_NAME}-php-fpm-ide
    image: php:{{PHP_VERSION}}-fpm
    volumes:
      - ./:/var/www
      - ~/.composer:/home/.composer
      - ~/.gitconfig:/home/.gitconfig
      - ~/.git-credentials:/home/.git-credentials
      - ~/.ssh:/home/.ssh
      - ~/.ssh/id_rsa:/home/.ssh/id_rsa
      - .devops/docker/php/custom.ini:/usr/local/etc/php/conf.d/custom.ini
    user: 1000:1000
    working_dir: /var/www
    profiles: ["ide"]
    networks:
      default:

  ### PHP-FPM ##############################################
  php-fpm:
    container_name: ${COMPOSE_PROJECT_NAME}-php-fpm
    build:
      context: .devops/docker
      dockerfile: Dockerfile
    volumes:
      - ./:/var/www
      - ~/.composer:/home/.composer
      - ~/.gitconfig:/home/.gitconfig
      - ~/.git-credentials:/home/.git-credentials
      - ~/.ssh:/home/.ssh
      - ~/.ssh/id_rsa:/home/.ssh/id_rsa
      - .devops/docker/php/custom.ini:/usr/local/etc/php/conf.d/custom.ini
    depends_on:
      - redis
    user: ${UID:-1000}:${GID:-1000}
    working_dir: /var/www
    environment:
      - PHP_IDE_CONFIG=serverName=${COMPOSE_PROJECT_NAME}.localhost
      - XDEBUG_MODE=${XDEBUG_MODE:-off}
    networks:
      default:
        aliases:
          - ${COMPOSE_PROJECT_NAME}-php-fpm
      app-network:

  ### Nginx Server ########################################
  nginx:
    container_name: ${COMPOSE_PROJECT_NAME}-nginx
    image: nginx:1.10.3
    volumes:
      - ./:/var/www
      - .devops/docker/nginx/default:/etc/nginx/sites-available/default
    environment:
      - VIRTUAL_HOST=${COMPOSE_PROJECT_NAME}.localhost
      - SELF_SIGNED_HOST=${COMPOSE_PROJECT_NAME}.localhost
      - PHP_HOST=${COMPOSE_PROJECT_NAME}-php-fpm
    depends_on:
      - php-fpm
    networks:
      default:
      proxy:
      app-network:
        aliases:
          - ${COMPOSE_PROJECT_NAME}-web

  ### MariaDB ##############################################
  db:
    hostname: ${DB_HOST}
    container_name: ${COMPOSE_PROJECT_NAME}-db-${DB_HOST}
    image: mariadb:{{DB_VERSION}}
    env_file: .env
    ports:
      - "3306:3306"
    volumes:
      - mysql-data:/var/lib/mysql
      - .devops/docker/mysql-init:/docker-entrypoint-initdb.d
    environment:
      - MYSQL_ROOT_PASSWORD=${COMPOSE_DB_ROOT_PASSWORD}
    networks:
      default:
      app-network:

  ### phpMyAdmin ###########################################
  pma:
    container_name: ${COMPOSE_PROJECT_NAME}-pma
    image: phpmyadmin/phpmyadmin:5.2
    environment:
      - VIRTUAL_HOST=pma-${COMPOSE_PROJECT_NAME}.localhost
      - SELF_SIGNED_HOST=pma-${COMPOSE_PROJECT_NAME}.localhost
      - PMA_HOST=${DB_HOST}
      - PMA_USER=${COMPOSE_DB_ROOT_USERNAME}
      - PMA_PASSWORD=${COMPOSE_DB_ROOT_PASSWORD}
    networks:
      - default
      - proxy
    depends_on:
      - php-fpm
    links:
      - db:${DB_HOST}

  ### Redis ################################################
  redis:
    container_name: ${COMPOSE_PROJECT_NAME}-redis
    image: redis:6
    ports:
      - "${REDIS_PORT:-6379}"
    networks:
      - default

  ### MailHog ################################################
  mailhog:
    container_name: ${COMPOSE_PROJECT_NAME}-mailhog
    image: mailhog/mailhog
    environment:
      - VIRTUAL_HOST=mailhog-${COMPOSE_PROJECT_NAME}.localhost
      - SELF_SIGNED_HOST=mailhog-${COMPOSE_PROJECT_NAME}.localhost
      - VIRTUAL_PORT=8025
    networks:
      - default
      - proxy

networks:
  proxy:
    external: true
  app-network:
    external: true

volumes:
  mysql-data:
```

## docker-compose.yml - Microservice / API (with Apache)

```yaml
services:

  ### PHP-FPM ##############################################
  php-fpm:
    hostname: ${COMPOSE_PROJECT_NAME}_php-fpm
    image: php:{{PHP_VERSION}}-fpm
    volumes:
      - ./:/var/www
      - ~/.composer:/home/.composer
      - ~/.gitconfig:/home/.gitconfig
      - ~/.git-credentials:/home/.git-credentials
      - ~/.ssh:/home/.ssh
      - ~/.ssh/id_rsa:/home/.ssh/id_rsa
      - ./.devops/php/custom.ini:/usr/local/etc/php/conf.d/custom.ini
    user: 1000:1000
    working_dir: /var/www
    networks:
      default:
        aliases:
          - ${COMPOSE_PROJECT_NAME}-php-fpm
      app-network:

  ### Apache Server ########################################
  apache:
    image: apache:2.4
    networks:
      default:
      proxy:
      app-network:
        aliases:
          - ${COMPOSE_PROJECT_NAME}-web
    volumes:
      - ./:/var/www
    links:
      - php-fpm:${COMPOSE_PROJECT_NAME}_php-fpm
    environment:
      - VIRTUAL_HOST=${COMPOSE_PROJECT_NAME}.localhost
      - SELF_SIGNED_HOST=${COMPOSE_PROJECT_NAME}.localhost
      - PHP_HOST=${COMPOSE_PROJECT_NAME}_php-fpm
      - DOCUMENT_ROOT=/var/www/public

  ### MariaDB ##############################################
  db:
    hostname: ${DB_HOST}
    image: mariadb:{{DB_VERSION}}
    networks:
      default:
        aliases:
          - ${DB_HOST}
          - ${DB_HOST}-testing
    volumes:
      - mysql-data:/var/lib/mysql
      - .devops/docker/mysql-initdb:/docker-entrypoint-initdb.d
    environment:
      - DATABASES=${DB_DATABASE} ${DB_DATABASE}_test
      - MYSQL_ROOT_PASSWORD=${DB_PASSWORD}

  ### Redis ################################################
  redis:
    container_name: ${COMPOSE_PROJECT_NAME}-redis
    image: redis:alpine
    networks:
      - default

  ### phpMyAdmin ###########################################
  pma:
    image: phpmyadmin/phpmyadmin:5.2
    networks:
      - default
      - proxy
    environment:
      - VIRTUAL_HOST=pma-${COMPOSE_PROJECT_NAME}.localhost
      - PMA_HOST=${DB_HOST}
      - PMA_USER=root
      - PMA_PASSWORD=${DB_PASSWORD}
      - SELF_SIGNED_HOST=pma-${COMPOSE_PROJECT_NAME}.localhost

volumes:
  mysql-data:

networks:
  proxy:
    external: true
  app-network:
    external: true
```

## .devops/docker/Dockerfile (Development - with Xdebug)

```dockerfile
FROM php:{{PHP_VERSION}}-fpm

# Installation des extensions PECL
RUN pecl install xdebug-3.4.3 \
    && docker-php-ext-enable xdebug

COPY xdebug/xdebug.ini /usr/local/etc/php/conf.d/xdebug.ini
```

## .devops/docker/php/custom.ini

```ini
opcache.enable=0
post_max_size = 5M
upload_max_filesize = 5M
memory_limit = 512M
```

## .devops/docker/php/xdebug.ini (monolith only)

```ini
[xdebug]

xdebug.mode=${XDEBUG_MODE}
xdebug.client_host=host.docker.internal
xdebug.client_port=9003
xdebug.output_dir=/var/www/storage/logs/xdebug
xdebug.log=/var/www/storage/logs/xdebug/xdebug.log
; auto start xdebug on every HTTP requests or on every CLI runs (if the IDE is listening)
xdebug.start_with_request=yes
```

## .devops/docker/xdebug/xdebug.ini

```ini
zend_extension=xdebug
xdebug.mode=develop,debug,coverage
```

## .devops/docker/nginx/default (monolith only)

```nginx
server {
	listen 80 default_server;
	listen [::]:80 default_server;

	root /var/www/public;

	index index.html index.htm index.nginx-debian.html;

	server_name _;

	location / {
		try_files $uri $uri/ /index.php?$query_string;
	}
	index index.php index.html index.htm;

	location ~ \.php$ {
        fastcgi_split_path_info ^(.+\.php)(/.*)$;
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_param SCRIPT_NAME $fastcgi_script_name;
        fastcgi_param PATH_INFO $fastcgi_path_info;
        fastcgi_param modHeadersAvailable true;
        fastcgi_param front_controller_active true;
        fastcgi_read_timeout 180;
        fastcgi_pass_header Authorization;
        fastcgi_pass {{PROJECT_NAME}}-php-fpm:9000;
        fastcgi_intercept_errors on;
        fastcgi_request_buffering off;
        fastcgi_param HTTPS on;
	}
}
```

## .devops/docker/mysql-init/create_dev.sh (monolith)

```bash
#!/bin/bash

# Init default database
init_default_database() {
    echo "
        CREATE DATABASE ${DB_DATABASE};
        GRANT ALL ON ${DB_DATABASE}.* TO ${DB_USERNAME}@'%' IDENTIFIED BY '${DB_PASSWORD}';
    " | mariadb -u ${COMPOSE_DB_ROOT_USERNAME} --password=${COMPOSE_DB_ROOT_PASSWORD}
}

# Main execution:
main() {
  init_default_database
}

# Executes the main routine with environment variables
main "$@"
```

## .devops/docker/mysql-initdb/001-init.sh (microservice/API)

```bash
#!/usr/bin/env sh

(
    set -eu

    for database in ${DATABASES}; do
        password="${database}"
        username="${database}"

        echo "Creating database ${database}..."

        echo "CREATE DATABASE IF NOT EXISTS \`${database}\` CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci ;GRANT ALL ON \`${database}\`.* TO '${username}'@'%' IDENTIFIED BY '${password}';" |
            mysql -u root --password=${MYSQL_ROOT_PASSWORD}
    done
)
```

## .devops/docker/mysql-init/z-flush.sql

```sql
FLUSH PRIVILEGES;
```
