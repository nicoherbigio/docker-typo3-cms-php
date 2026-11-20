FROM debian:trixie-slim

LABEL maintainer="Nico Herbig <nico.herbig@whz.de>"


RUN set -eux; \
    apt-get update; \
    apt-get -y install lsb-release ca-certificates curl; \
    curl -sSLo /tmp/debsuryorg-archive-keyring.deb https://packages.sury.org/debsuryorg-archive-keyring.deb; \
    dpkg -i /tmp/debsuryorg-archive-keyring.deb; \
    sh -c 'echo "deb [signed-by=/usr/share/keyrings/debsuryorg-archive-keyring.gpg] https://packages.sury.org/php/ $(lsb_release -sc) main" > /etc/apt/sources.list.d/php.list'; \
    apt-get update; \
    \
	  apt-get install -y --no-install-recommends apache2; \
    a2dismod mpm_event && a2enmod mpm_prefork; \
    \
    rm -rvf /var/www/html/*; \
	  \
    ln -sfT /dev/stderr "/var/log/apache2/error.log"; \
    ln -sfT /dev/stdout "/var/log/apache2/access.log"; \
    ln -sfT /dev/stdout "/var/log/apache2/other_vhosts_access.log"; \
    chown -R --no-dereference "www-data:www-data" "/var/log/apache2"; \
    \
    apt-get install -y --no-install-recommends libapache2-mod-php7.4 php7.4-common php7.4-apcu php7.4-curl php7.4-gd php7.4-intl php7.4-ldap php7.4-mbstring php7.4-mysqli php7.4-opcache php7.4-soap php7.4-xml php7.4-zip; \
    apt-get install -y --no-install-recommends ghostscript graphicsmagick; \
    \
    { \
        echo '<VirtualHost *:80>'; \
        echo '\tServerAdmin webmaster@localhost'; \
        echo '\tDocumentRoot /var/www/html'; \
        echo ''; \
        echo '\tErrorLog ${APACHE_LOG_DIR}/error.log'; \
        echo '\tCustomLog ${APACHE_LOG_DIR}/access.log combined'; \
        echo ''; \
        echo '\t<Directory /var/www/html>'; \
        echo '\t\tOptions -Indexes'; \
        echo '\t\tAllowOverride All'; \
        echo '\t</Directory>'; \
        echo '</VirtualHost>'; \
    } | tee "/etc/apache2/sites-available/000-default.conf"; \
    \
    rm -rf /var/lib/apt/lists/*


# Apache + PHP requires preforking Apache for best results
RUN a2dismod mpm_event && a2enmod mpm_prefork


# #############################################################################


# Add additional Apache user and add them to the Apache group (www-data) to ensure that the ID (UID)
# is consistently assigned regardless of which additional dependencies are added.

ENV WWW_USER_ID=10033 WWW_USER=www-user
ENV WWW_GROUP_ID=33 WWW_GROUP=www-data

RUN useradd --gid ${WWW_GROUP} --uid ${WWW_USER_ID} --create-home --shell /bin/bash ${WWW_USER}


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
       } | tee /etc/php/7.4/apache2/conf.d/docker-php-timezone.ini \
    && { \
        echo 'date.timezone = ${TZ}'; \
       } | tee /etc/php/7.4/cli/conf.d/docker-php-timezone.ini


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

ENV LANG=de_DE.UTF-8 LANGUAGE=de_DE:de


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
    && sed -ri -e 's!/var/www/!${APACHE_DOCUMENT_ROOT}!g' /etc/apache2/apache2.conf /etc/apache2/conf-available/*.conf /etc/apache2/mods-available/*.conf \
    && { \
        echo 'max_execution_time = 240'; \
        echo 'max_input_vars = 1500'; \
        echo 'memory_limit = 512M'; \
        echo 'post_max_size = 256M'; \
        echo 'upload_max_filesize = 256M'; \
       } | tee /etc/php/7.4/apache2/conf.d/docker-typo3-cms.ini \
    && { \
        echo 'max_execution_time = 240'; \
        echo 'max_input_vars = 1500'; \
        echo 'memory_limit = 512M'; \
        echo 'post_max_size = 256M'; \
        echo 'upload_max_filesize = 256M'; \
       } | tee /etc/php/7.4/cli/conf.d/docker-typo3-cms.ini

WORKDIR ${APACHE_DOCUMENT_ROOT}


# #############################################################################


COPY --chmod=755 assets/docker-php-entrypoint /usr/local/bin/
COPY --chmod=755 assets/apache2-foreground /usr/local/bin/

# https://httpd.apache.org/docs/2.4/stopping.html#gracefulstop
STOPSIGNAL SIGWINCH

EXPOSE 80

ENTRYPOINT ["docker-php-entrypoint"]
CMD ["apache2-foreground"]
