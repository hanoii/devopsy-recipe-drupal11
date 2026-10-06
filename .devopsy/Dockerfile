# syntax=docker/dockerfile:1

# Drupal on FrankenPHP (Caddy + PHP 8.4), built on the server by
# `devopsy @<target> release deploy` (mode: build). Code is owned by root and
# read-only; the only writable path at runtime is /app/storage, a bind mount
# of .devopsy/mnt/storage (see docker/entrypoint).
#
# The build context is the project root (see compose.yaml), filtered by
# Dockerfile.dockerignore next to this file.

########################################
# Base: PHP runtime shared by build and app
########################################
FROM dunglas/frankenphp:1-php8.4-trixie AS base

RUN install-php-extensions apcu gd intl opcache pdo_mysql zip

# curl for the healthcheck, mariadb-client for drush sql:*. No recommends, and
# the apt lists are removed in the same layer to keep the image small.
RUN apt-get update \
  && apt-get install -y --no-install-recommends \
    curl \
    mariadb-client \
  && rm -rf /var/lib/apt/lists/*

########################################
# Build: composer install
########################################
FROM base AS build

COPY --from=composer:2 /usr/bin/composer /usr/local/bin/composer
ENV COMPOSER_ALLOW_SUPERUSER=1

WORKDIR /app

# Only what composer needs, so vendor/ and web/core stay cached until
# dependencies change.
COPY composer.json composer.lock ./
RUN composer install --no-dev --optimize-autoloader --no-interaction --no-progress

# Then the rest of the project (settings, config, custom code) on top.
COPY . .

########################################
# Runtime
########################################
FROM base AS app

# Not root, and not 1000: that is the devopsy deploy user on servers. See
# "Conventions" in the devopsy workspace README.
ARG APP_UID=10001

# The root filesystem is read-only at runtime: Caddy keeps its data and config
# in /tmp, a tmpfs (see compose.yaml).
ENV STORAGE_DIR=/app/storage \
  XDG_CONFIG_HOME=/tmp/caddy/config \
  XDG_DATA_HOME=/tmp/caddy/data

RUN useradd --uid ${APP_UID} --user-group --no-create-home --home-dir /tmp --shell /bin/bash app \
  && mv "$PHP_INI_DIR/php.ini-production" "$PHP_INI_DIR/php.ini"

COPY .devopsy/docker/Caddyfile /etc/frankenphp/Caddyfile
COPY .devopsy/docker/php.ini "$PHP_INI_DIR/conf.d/zz-app.ini"
COPY --chmod=755 .devopsy/docker/entrypoint /usr/local/bin/entrypoint
# Keep /app/vendor/bin out of PATH: its drush would win over this wrapper and
# run as root from `docker compose exec`.
COPY --chmod=755 .devopsy/docker/drush /usr/local/bin/drush

# Normalize permissions while copying (no extra layer): the build context
# keeps the host's modes, and the app user must be able to read everything but
# write nothing.
COPY --from=build --chmod=u=rwX,go=rX /app /app

# Public files live in the storage mount, next to private files.
RUN rm -rf /app/web/sites/default/files \
  && ln -s ${STORAGE_DIR}/public /app/web/sites/default/files

WORKDIR /app
EXPOSE 8080

ENTRYPOINT ["entrypoint"]
CMD ["frankenphp", "run", "--config", "/etc/frankenphp/Caddyfile", "--adapter", "caddyfile"]
