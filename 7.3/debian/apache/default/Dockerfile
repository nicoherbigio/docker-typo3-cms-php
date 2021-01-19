FROM nicoherbigio/php:7.3.26-apache

LABEL maintainer="Nico Herbig <nico@nicoherbig.io>"


# Add additional Apache user and add them to the Apache group (www-data) to ensure that the ID (UID)
# is consistently assigned regardless of which additional dependencies are added.

ENV WWW_USER_ID=10033 WWW_USER=www-user
ENV WWW_GROUP_ID=33 WWW_GROUP=www-data

RUN useradd --gid ${WWW_GROUP} --uid ${WWW_USER_ID} --create-home --shell /bin/bash ${WWW_USER}


# Install additional required software packages for TYPO3 CMS.

RUN set -ex \
    && apt-get update \
    && apt-get install -y --no-install-recommends git libfreetype6-dev libjpeg62-turbo-dev libmcrypt-dev libpng-dev libxml2-dev libzip-dev libldap2-dev ghostscript graphicsmagick unzip wget \
    && docker-php-ext-configure gd --with-freetype-dir=/usr/include/ --with-jpeg-dir=/usr/include/ \
    && docker-php-ext-install -j$(nproc) exif gd intl ldap mysqli opcache soap zip \
    && pecl install apcu-5.1.18 \
    && docker-php-ext-enable apcu \
    && apt-get remove -y libfreetype6-dev libjpeg62-turbo-dev libmcrypt-dev libpng-dev libxml2-dev libzip-dev \
    && rm -rf /var/lib/apt/lists/*


# Reconfigure Apache and PHP for TYPO3 CMS.

RUN set -ex \
    && a2enmod expires deflate headers rewrite \
    && { \
        echo 'max_execution_time = 240'; \
        echo 'max_input_vars = 1500'; \
        echo 'memory_limit = 512M'; \
        echo 'post_max_size = 32M'; \
        echo 'upload_max_filesize = 32M'; \
       } | tee /usr/local/etc/php/conf.d/docker-typo3-cms.ini
