ARG DOCKER_HUB_PROXY=""

FROM "${DOCKER_HUB_PROXY}debian:bullseye-slim" as composer-build

    ENV DEBIAN_FRONTEND noninteractive
    ENV COMPOSER_ALLOW_SUPERUSER 1
    ARG MISP_TAG
    ARG MISP_COMMIT
    RUN --mount=type=cache,target=/var/cache/apt,sharing=locked \
        --mount=type=cache,target=/var/lib/apt,sharing=locked \
        rm -f /etc/apt/apt.conf.d/docker-clean; echo 'Binary::apt::APT::Keep-Downloaded-Packages "true";' > /etc/apt/apt.conf.d/keep-cache; \
        apt-get update; apt-get install -y --no-install-recommends \
        ca-certificates \
        php \
        php-apcu \
        php-curl \
        php-xml \
        php-intl \
        php-bcmath \
        php-mbstring \
        php-mysql \
        php-redis \
        php-gd \
        php-fpm \
        php-zip \
        unzip
    
    WORKDIR /tmp
    ADD https://raw.githubusercontent.com/MISP/MISP/${MISP_COMMIT:-${MISP_TAG}}/app/composer.json /tmp
    COPY --from=composer:latest /usr/bin/composer /usr/bin/composer
    RUN composer config --no-interaction allow-plugins.composer/installers true
    RUN composer install
    RUN composer require --with-all-dependencies --no-interaction \
            supervisorphp/supervisor:^4.0 \
            guzzlehttp/guzzle \
            lstrojny/fxmlrpc \
            php-http/message \
            php-http/message-factory \
            # docker image specific dependencies
            elasticsearch/elasticsearch:^8.7.0 \
            jakub-onderka/openid-connect-php:^1.0.0 \
            aws/aws-sdk-php

FROM "${DOCKER_HUB_PROXY}debian:bullseye-slim" as php-build
    ENV DEBIAN_FRONTEND noninteractive
    ENV TZ Etc/UTC
    RUN --mount=type=cache,target=/var/cache/apt,sharing=locked \
        --mount=type=cache,target=/var/lib/apt,sharing=locked \
        rm -f /etc/apt/apt.conf.d/docker-clean; echo 'Binary::apt::APT::Keep-Downloaded-Packages "true";' > /etc/apt/apt.conf.d/keep-cache; \
        apt-get update; apt-get install -y --no-install-recommends \
        gcc \
        g++ \
        make \
        libfuzzy-dev \
        ca-certificates \
        php \
        php-dev \
        php-xml \
        php-pear \
        librdkafka-dev \
        libsimdjson-dev \
        git
    RUN cp "/usr/lib/$(gcc -dumpmachine)"/libfuzzy.* /usr/lib; pecl channel-update pecl.php.net && pecl install ssdeep && pecl install rdkafka && pecl install simdjson
    RUN git clone --recursive --depth=1 https://github.com/kjdev/php-ext-brotli.git && \
        cd php-ext-brotli && phpize && ./configure && make && make install

FROM "${DOCKER_HUB_PROXY}debian:bullseye-slim" as python-build
    ENV DEBIAN_FRONTEND noninteractive

    RUN --mount=type=cache,target=/var/cache/apt,sharing=locked \
        --mount=type=cache,target=/var/lib/apt,sharing=locked \
        rm -f /etc/apt/apt.conf.d/docker-clean; echo 'Binary::apt::APT::Keep-Downloaded-Packages "true";' > /etc/apt/apt.conf.d/keep-cache; \
        apt-get update; apt-get install -y --no-install-recommends python3-pip git
    ARG MISP_TAG
    ARG MISP_COMMIT
    # Download MISP using git in the /var/www/ directory. Remove unnecessary items.
    RUN <<-EOF
        if [ ! -z "${MISP_COMMIT}" ]; then
            git clone https://github.com/MISP/MISP.git /var/www/MISP && cd /var/www/MISP && git checkout "${MISP_COMMIT}"
        else
            git clone --branch "${MISP_TAG}" --depth 1 https://github.com/MISP/MISP.git /var/www/MISP
        fi

        cd /var/www/MISP || exit; git submodule update --init --recursive .

        # Until MISP project specifies its required PyMISP version, we grab this from the submodule
        pymispversion=$(head -n1 PyMISP/pymisp/__init__.py | awk '{print $3}' | tr -d \')

        if [ -n "$pymispversion" ]; then
            sed -i "s/pymisp$/pymisp==$pymispversion/" requirements.txt
            cat requirements.txt
        fi
        # End of pymisp version fixing hack

        # Remove modules only used during MISP build
        set -- 'coveralls' 'codecov' 'requests-mock' 'nose' 'pip'
        for mod in "$@"; do
            sed "/${mod}/d" -i /var/www/MISP/requirements.txt
        done;

        # Add additional dependencies (container specific)
        set -- "redis${PYPI_REDIS_VERSION}" "lief${PYPI_LIEF_VERSION}" "pydeep2${PYPI_PYDEEP2_VERSION}" "python-magic${PYPI_PYTHON_MAGIC_VERSION}" "misp-lib-stix2${PYPI_MISP_LIB_STIX2_VERSION}" "maec${PYPI_MAEC_VERSION}" "mixbox${PYPI_MIXBOX_VERSION}" "cybox${PYPI_CYBOX_VERSION}"
        for mod in "$@"; do
            grep -qxF ${mod} /var/www/MISP/requirements.txt || echo ${mod} >> /var/www/MISP/requirements.txt
        done;

        pip3 wheel --no-cache-dir -w /wheels/ -r /var/www/MISP/requirements.txt
        
        # Remove files we do not care for
        rm -r /var/www/MISP/PyMISP
        find /var/www/MISP/INSTALL/* ! -name 'MYSQL.sql' -type f -exec rm {} +
        find /var/www/MISP/INSTALL/* ! -name 'MYSQL.sql' -type l -exec rm {} +
        rm -rf /var/www/MISP/.git

        # This is PHP related but useful to do in the same step as checkout of MISP
        mkdir -p /var/www/MISP/app/Vendor && mkdir -p /var/www/MISP/app/Plugin
EOF

FROM "${DOCKER_HUB_PROXY}debian:bullseye-slim"

    ENV DEBIAN_FRONTEND noninteractive
    ARG MISP_TAG
    ARG MISP_COMMIT
    ARG PHP_VER
    RUN --mount=type=cache,target=/var/cache/apt,sharing=locked \
        --mount=type=cache,target=/var/lib/apt,sharing=locked \
        rm -f /etc/apt/apt.conf.d/docker-clean; echo 'Binary::apt::APT::Keep-Downloaded-Packages "true";' > /etc/apt/apt.conf.d/keep-cache; \
        apt-get update; apt-get install -y --no-install-recommends \
        procps \
        sudo \
        nginx \
        supervisor \
        cron \
        openssl \
        gpg \
        gpg-agent \
        ssdeep \
        libfuzzy2 \
        mariadb-client \
        rsync \
        # Python Requirements
        python3 \
        python3-setuptools \
        python3-pip \
        # PHP Requirements
        php \
        php-apcu \
        php-curl \
        php-xml \
        php-intl \
        php-bcmath \
        php-mbstring \
        php-mysql \
        php-redis \
        php-gd \
        php-fpm \
        php-zip \
        librdkafka1 \
        libbrotli1 \
        libsimdjson5 \
        # Unsure we need these
        zip unzip \
        # Require for advanced an unattended configuration
        curl \
        jq

    # Install python modules from cache written to by python-build
    RUN --mount=type=bind,from=python-build,source=/wheels,target=/wheels \
        pip3 install --no-cache-dir /wheels/*.whl

    # PHP: install prebuilt libraries, then install the app's PHP deps
    COPY --from=php-build ["/usr/lib/php/${PHP_VER}/ssdeep.so", "/usr/lib/php/${PHP_VER}/rdkafka.so", "/usr/lib/php/${PHP_VER}/brotli.so", "/usr/lib/php/${PHP_VER}/simdjson.so", "/usr/lib/php/${PHP_VER}/"]

    # Do an early chown to limit image size
    COPY --from=python-build --chown=www-data:www-data --chmod=0550 /var/www/MISP /var/www/MISP
    COPY --from=composer-build --chown=www-data:www-data --chmod=0550 /tmp/Vendor /var/www/MISP/app/Vendor
    COPY --from=composer-build --chown=www-data:www-data --chmod=0550 /tmp/Plugin /var/www/MISP/app/Plugin

    # Gather these in one layer, only act on actual directories under /etc/php/
    RUN <<-EOF
        set -- "ssdeep" "rdkafka" "brotli" "simdjson"
        for mod in "$@"; do
            for dir in /etc/php/*/; do
                echo "extension=${mod}.so" > "${dir}mods-available/${mod}.ini"
            done;
            phpenmod "${mod}"
        done;
        phpenmod redis
EOF

    # nginx
    RUN rm /etc/nginx/sites-enabled/*; mkdir /run/php /etc/nginx/certs

    # Make a copy of the file and configuration stores, so we can sync from it

    # The spirit of the upstream dockerization is to make:
    #   1) User and group aligned in terms of permissions
    #   2) Files executable and read only, because of some rogue scripts like 'cake'
    #   3) Directories writable, because sometimes MISP add new new files

    RUN <<-EOF
        cp -R /var/www/MISP/app/files /var/www/MISP/app/files.dist
        cp -R /var/www/MISP/app/Config /var/www/MISP/app/Config.dist
        find /var/www/MISP \( ! -user www-data -or ! -group www-data \) -exec chown www-data:www-data '{}' +;
        find /var/www/MISP -not -perm 550 -type f -exec chmod 0550 '{}' +;
        find /var/www/MISP -not -perm 770 -type d -exec chmod 0770 '{}' +;
EOF
        
    # Copy all our image specific files to appropriate locations
    COPY files/ /
    ENTRYPOINT [ "/entrypoint.sh" ]

    # Change Workdirectory
    WORKDIR /var/www/MISP
