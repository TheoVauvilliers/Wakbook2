ARG PHP_VERSION=8.4

# ---- BASE ----
FROM php:${PHP_VERSION}-apache AS base
ENV APACHE_DOCUMENT_ROOT=/var/www/html/public

RUN apt-get update && apt-get install -y \
    git unzip libicu-dev libzip-dev libpng-dev libjpeg-dev libfreetype6-dev \
    libxml2-dev libpq-dev wget openssl curl \
 && docker-php-ext-install intl opcache exif zip pdo pdo_pgsql \
 && docker-php-ext-enable exif pdo_pgsql \
 && rm -rf /var/lib/apt/lists/*

COPY --from=composer:2 /usr/bin/composer /usr/bin/composer
ENV COMPOSER_ALLOW_SUPERUSER=1

RUN wget https://get.symfony.com/cli/installer -O - | bash \
 && mv /root/.symfony*/bin/symfony /usr/local/bin/symfony

RUN sed -ri -e 's!/var/www/html!${APACHE_DOCUMENT_ROOT}!g' /etc/apache2/sites-available/*.conf \
 && sed -ri -e 's!/var/www/!${APACHE_DOCUMENT_ROOT}!g' /etc/apache2/apache2.conf /etc/apache2/conf-available/*.conf \
 && a2enmod rewrite headers

WORKDIR /var/www/html

# ---- DEV ----
FROM base AS dev

RUN pecl install xdebug && docker-php-ext-enable xdebug
COPY docker/phpapache/conf.d/xdebug.ini /usr/local/etc/php/conf.d/xdebug.ini
RUN set -eux; \
  echo '#!/bin/bash' > /usr/local/bin/xdebug; \
  echo 'set -e' >> /usr/local/bin/xdebug; \
  echo 'XDEBUG_INI="/usr/local/etc/php/conf.d/docker-php-ext-xdebug.ini"' >> /usr/local/bin/xdebug; \
  echo 'if [ -f "$XDEBUG_INI" ]; then rm -f "$XDEBUG_INI"; pkill -o -USR2 php-fpm 2>/dev/null || apachectl restart; else docker-php-ext-enable xdebug >/dev/null; pkill -o -USR2 php-fpm 2>/dev/null || apachectl restart; fi' >> /usr/local/bin/xdebug; \
  chmod +x /usr/local/bin/xdebug

ENV NVM_DIR=/home/phpuser/.nvm
ENV NODE_VERSION=20
RUN mkdir -p $NVM_DIR \
 && curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.6/install.sh | bash \
 && bash -c "source $NVM_DIR/nvm.sh && nvm install $NODE_VERSION && nvm alias default $NODE_VERSION && npm install -g yarn"

RUN echo 'export NVM_DIR="$HOME/.nvm"' >> /home/phpuser/.bashrc \
 && echo '[ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"' >> /home/phpuser/.bashrc \
 && echo '[ -s "$NVM_DIR/bash_completion" ] && \. "$NVM_DIR/bash_completion"' >> /home/phpuser/.bashrc

EXPOSE 80 443

# ---- PROD ----
FROM base AS prod

COPY composer.json composer.lock* symfony.lock* ./

RUN composer install --no-dev --no-interaction --prefer-dist --no-progress --no-scripts

COPY . .

RUN composer dump-autoload --no-dev --classmap-authoritative

#RUN npm ci && npm run build

EXPOSE 80
CMD ["apache2-foreground"]
