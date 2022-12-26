FROM ubuntu:jammy as base

LABEL MAINTAINER="GoldElysium <goldelysium@hlresort.community>"

ENV DEBIAN_FRONTEND="noninteractive" \
    TZ="Europe/London"

RUN \
    apt-get update && \
    apt-get install -y curl gettext nginx php php-bcmath php-common php-dom php-fileinfo \
    php-fpm php-gd php-memcached php-mysql php-mbstring php-pdo php-phar php-json \
    php-simplexml php-tokenizer php-ctype php-curl php-zip php-xmlwriter \
    tini gnupg unzip netcat \
    && rm -rf /var/www/html \
    && mkdir -p /var/www/html /run/nginx /etc/nginx/conf.d/

RUN curl -fsSL https://deb.nodesource.com/setup_16.x | bash - \
    && apt-get install -y nodejs

# Build phase of the container
# This is where composer is added and pterodactyl properly setup
FROM base as build

ARG VERSION
ARG GIT_BRANCH=release/${VERSION}

ENV VERSION=${VERSION}

WORKDIR /var/www/html

RUN \
    curl -sS https://dl.yarnpkg.com/debian/pubkey.gpg | apt-key add - \
    && echo "deb https://dl.yarnpkg.com/debian/ stable main" | tee /etc/apt/sources.list.d/yarn.list \
    && apt-get update \
    && apt-get install -y yarn git

RUN \
    git clone https://github.com/pterodactyl/panel ./ --depth 1 --branch ${GIT_BRANCH}; \
    rm .git -rf; \
    chmod -R 755 storage/* bootstrap/cache; \
    find storage -type d > .storage.tmpl; \
    curl -sS https://getcomposer.org/installer | php -- --install-dir=/usr/local/bin --filename=composer; \
    cp .env.example .env; \
    composer install --ansi --no-dev --optimize-autoloader; \
    chown -R www-data:www-data *;

RUN \
    yarn install --production; \
    yarn add cross-env; \
    yarn run build:production; \
    rm -rf node_modules;

# Remove persistent data folders as they will be symlinked further down
RUN rm .env ./storage -rf

# Final Production phase of the controller
# All build requirements get scrapped as to maintain a small image
FROM base as production

ARG VERSION

ENV VERSION=${VERSION}

WORKDIR /var/www/html

COPY --from=build --chown=www-data:www-data /var/www /var/www
COPY ./root/ /

# Symlink storage, config and cache to /data
RUN \
    ln -s /data/storage storage; \
    ln -s /data/pterodactyl.conf .env;

EXPOSE 80 443
VOLUME [ "/data" ]

ENTRYPOINT [ "/usr/bin/tini", "--", "/entrypoint.sh" ]
CMD [ "p:start" ]
