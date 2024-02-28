# syntax=docker/dockerfile:1.4

# Adapted from https://github.com/dunglas/symfony-docker

# Versions
# hadolint ignore=DL3007
FROM dunglas/frankenphp:latest-alpine AS frankenphp_upstream

# Base FrankenPHP image
FROM frankenphp_upstream AS frankenphp_base

WORKDIR /app

# persistent / runtime deps
# hadolint ignore=DL3018
RUN apk add --no-cache \
    acl \
    file \
    gettext \
    git \
    && apk add --no-cache --virtual .build-deps curl

# Install Composer dependencies
ENV COMPOSER_ALLOW_SUPERUSER=1
RUN set -eux; \
    install-php-extensions \
        @composer \
        apcu \
        intl \
        opcache \
        zip \
        pdo_pgsql \
    && apk del .build-deps

# Install Node.js and npm
RUN set -eux; \
    apk add --no-cache nodejs-current=21.4.0-r0 npm=10.2.5-r0 || (sleep 10 && apk add --no-cache nodejs-current=21.4.0-r0 npm=10.2.5-r0)

# Copy configuration files and entrypoint script
COPY --link frankenphp/conf.d/app.ini $PHP_INI_DIR/conf.d/
COPY --link --chmod=755 frankenphp/docker-entrypoint.sh /usr/local/bin/docker-entrypoint
COPY --link frankenphp/Caddyfile /etc/caddy/Caddyfile

ENTRYPOINT ["docker-entrypoint"]

HEALTHCHECK --start-period=60s CMD curl -f http://localhost:2019/metrics || exit 1
CMD [ "frankenphp", "run", "--config", "/etc/caddy/Caddyfile" ]

# Dev FrankenPHP image
FROM frankenphp_base AS frankenphp_dev

ENV APP_ENV=dev XDEBUG_MODE=off
VOLUME /app/var/

# Enable Xdebug for development
RUN mv "$PHP_INI_DIR/php.ini-development" "$PHP_INI_DIR/php.ini" \
    && install-php-extensions xdebug

COPY --link frankenphp/conf.d/app.dev.ini $PHP_INI_DIR/conf.d/

CMD [ "frankenphp", "run", "--config", "/etc/caddy/Caddyfile", "--watch" ]

# Prod FrankenPHP image
FROM frankenphp_base AS frankenphp_prod

ENV APP_ENV=prod

# Configure PHP for production
RUN mv "$PHP_INI_DIR/php.ini-production" "$PHP_INI_DIR/php.ini" \
    && COPY --link frankenphp/conf.d/app.prod.ini $PHP_INI_DIR/conf.d/ \
    && COPY --link frankenphp/worker.Caddyfile /etc/caddy/worker.Caddyfile

# Install PHP dependencies
COPY --link composer.* symfony.* ./
RUN set -eux; \
    composer install --no-cache --prefer-dist --no-dev --no-autoloader --no-scripts --no-progress \
    && rm -Rf frankenphp/ \
    && mkdir -p var/cache var/log \
    && composer dump-autoload --classmap-authoritative --no-dev \
    && composer dump-env prod \
    && composer run-script --no-dev post-install-cmd \
    && chmod +x bin/console; sync

# Install Node.js and npm (Production image)
RUN set -eux; \
    apk add --no-cache "nodejs-current=21.4.0-r0" "npm=10.2.5-r0" || (sleep 10 && apk add --no-cache "nodejs-current=21.4.0-r0" "npm=10.2.5-r0")

