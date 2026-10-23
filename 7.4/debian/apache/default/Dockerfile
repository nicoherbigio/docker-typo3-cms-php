FROM php:7.4.33-apache-bullseye AS builder

LABEL maintainer="Josh Bowen"
LABEL source="https://joshbowen.net/posts/patching-php-74-for-php2shell/"


# "Avoid state corruption via bailout" on php-src's PHP-7.4-CVE-2026-109618-109619
ARG PHP_FIX_COMMIT=712e60f9edfd03f0960fc4fc3c2c375f42b90aa2

RUN set -eux; \
  { \
    echo 'deb http://archive.debian.org/debian bullseye main'; \
    echo 'deb http://archive.debian.org/debian bullseye-updates main'; \
    echo 'deb http://archive.debian.org/debian-security bullseye-security main'; \
  } > /etc/apt/sources.list; \
  apt-get update; \
  apt-get install -y --no-install-recommends \
    $PHPIZE_DEPS git patch \
    libargon2-dev libcurl4-openssl-dev libonig-dev libreadline-dev libsodium-dev \
    libsqlite3-dev libssl-dev libxml2-dev zlib1g-dev \
  ; \
  case "$(php-config --configure-options)" in \
    *--with-apxs2*) apt-get install -y --no-install-recommends apache2-dev ;; \
  esac


RUN set -eux; \
  git init -q /usr/src/php-fix; \
  cd /usr/src/php-fix; \
  git remote add origin https://github.com/php/php-src.git; \
  git fetch -q --depth 2 --filter=blob:none origin "$PHP_FIX_COMMIT"; \
  git diff "$PHP_FIX_COMMIT^" "$PHP_FIX_COMMIT" > /usr/src/php-fix.patch


RUN set -eux; \
  export CFLAGS="$PHP_CFLAGS" CPPFLAGS="$PHP_CPPFLAGS" LDFLAGS="$PHP_LDFLAGS"; \
  configureOptions="$(php-config --configure-options)"; \
  docker-php-source extract; \
  cd /usr/src/php; \
  patch -p1 --forward --no-backup-if-mismatch < /usr/src/php-fix.patch; \
  \
  mkdir -p /out/usr/src; \
  tar -cJf /out/usr/src/php.tar.xz -C /usr/src --exclude=.docker-delete-me \
    --transform 's,^php,php-7.4.33,' php; \
  \
  debMultiarch="$(dpkg-architecture --query DEB_BUILD_MULTIARCH)"; \
  [ -e /usr/include/curl ] || [ -e /usr/local/include/curl ] \
    || ln -sT "/usr/include/$debMultiarch/curl" /usr/local/include/curl; \
  ./configure $configureOptions; \
  make -j "$(nproc)"; \
  mkdir -p /out/etc/apache2/mods-available; \
  make INSTALL_ROOT=/out install-sapi install-binaries install-build \
    install-headers install-programs install-modules; \
  rm -rf /out/etc; \
  find /out -type f -perm '/0111' -exec strip --strip-all '{}' + || :


# Build the final image
# #############################################################################

FROM php:7.4.33-apache-bullseye

LABEL maintainer="Nico Herbig <nico@nicoherbig.io>"


# Add additional Apache user and add them to the Apache group (www-data) to ensure that the ID (UID)
# is consistently assigned regardless of which additional dependencies are added.

ENV WWW_USER_ID=10033 WWW_USER=www-user
ENV WWW_GROUP_ID=33 WWW_GROUP=www-data

RUN useradd --gid ${WWW_GROUP} --uid ${WWW_USER_ID} --create-home --shell /bin/bash ${WWW_USER}


# Switch to the Debian archive repositories to ensure that all packages are available for installation.

RUN set -eux; \
  { \
    echo 'deb http://archive.debian.org/debian bullseye main'; \
    echo 'deb http://archive.debian.org/debian bullseye-updates main'; \
    echo 'deb http://archive.debian.org/debian-security bullseye-security main'; \
  } > /etc/apt/sources.list; \
  apt-get update; \
  apt-get upgrade -y; \
  rm -rf /var/lib/apt/lists/*


# Copy the patched PHP source and the compiled binaries from the builder stage into the final image.

COPY --from=builder /out/ /

RUN set -eux; \
  rm -f /usr/src/php.tar.xz.asc


# Set time zone to "Europe/Berlin".

ENV TZ=Europe/Berlin

RUN set -ex \
    && apt-get update \
    && apt-get install -y --no-install-recommends tzdata \
    && rm -rf /var/lib/apt/lists/* \
    && ln -fns /usr/share/zoneinfo/${TZ} /etc/localtime \
    && echo ${TZ} > /etc/timezone \
    && { \
        echo 'date.timezone = ${TZ}'; \
       } | tee /usr/local/etc/php/conf.d/docker-php-timezone.ini


# Set locale to "de_DE.UTF-8".

RUN set -ex \
    && if [ -f /etc/dpkg/dpkg.cfg.d/docker ]; then \
            # If this file exists, we are likely in "debian:xxx-slim" and locales are thus being excluded
            # so we need to remove that exclusion (since we need locales).
            grep -q '/usr/share/locale' /etc/dpkg/dpkg.cfg.d/docker; \
            sed -ri '/\/usr\/share\/locale/d' /etc/dpkg/dpkg.cfg.d/docker; \
            ! grep -q '/usr/share/locale' /etc/dpkg/dpkg.cfg.d/docker; \
       fi \
    && apt-get update \
    && apt-get install -y --no-install-recommends locales \
    && rm -rf /var/lib/apt/lists/* \
    && localedef -i de_DE -c -f UTF-8 -A /usr/share/locale/locale.alias de_DE.UTF-8 \
    && localedef -i en_US -c -f UTF-8 -A /usr/share/locale/locale.alias en_US.UTF-8

ENV LANG=en_US.UTF-8 LANGUAGE=en_US:en


# Install additional required software packages for TYPO3 CMS.

RUN set -ex \
    && apt-get update \
    && apt-get install -y --no-install-recommends git libfreetype6-dev libjpeg62-turbo-dev libmcrypt-dev libpng-dev libxml2-dev libzip-dev libldap2-dev ghostscript graphicsmagick unzip wget \
    && docker-php-ext-configure gd --with-freetype --with-jpeg \
    && docker-php-ext-install -j$(nproc) exif gd intl ldap mysqli opcache soap zip \
    && pecl install apcu-5.1.21 \
    && docker-php-ext-enable apcu \
    && apt-get remove -y libfreetype6-dev libjpeg62-turbo-dev libmcrypt-dev libpng-dev libxml2-dev libzip-dev \
    && rm -rf /var/lib/apt/lists/*


# Configure Apache and PHP for TYPO3 CMS.
#   - activate required Apache modules for TYPO3 CMS
#   - switch Apache server signature off
#   - make Apache document root configureable by environment variable
#   - use PHP production settings as default
#   - reconfigure PHP settings for TYPO3 CMS

ENV APACHE_DOCUMENT_ROOT=/app/public

RUN set -ex \
    && a2enmod expires deflate headers rewrite \
    && sed -i -e '/ServerTokens/s/OS/Prod/g' -e '/ServerSignature/s/On/Off/g' /etc/apache2/conf-available/security.conf \
    && sed -ri -e 's!/var/www/html!${APACHE_DOCUMENT_ROOT}!g' /etc/apache2/sites-available/*.conf \
    && sed -ri -e 's!/var/www/!${APACHE_DOCUMENT_ROOT}!g' /etc/apache2/apache2.conf /etc/apache2/conf-available/*.conf \
    && cp ${PHP_INI_DIR}/php.ini-production ${PHP_INI_DIR}/php.ini \
    && { \
        echo 'max_execution_time = 240'; \
        echo 'max_input_vars = 1500'; \
        echo 'memory_limit = 512M'; \
        echo 'post_max_size = 256M'; \
        echo 'upload_max_filesize = 256M'; \
       } | tee /usr/local/etc/php/conf.d/docker-typo3-cms.ini


WORKDIR /app
