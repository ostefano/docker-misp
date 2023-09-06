ARG DOCKER_HUB_PROXY=""

FROM "${DOCKER_HUB_PROXY}debian:bullseye-slim" as composer-build

    ENV DEBIAN_FRONTEND noninteractive
    ENV COMPOSER_ALLOW_SUPERUSER 1
    ARG MISP_TAG
    ARG MISP_COMMIT
    RUN apt-get update; apt-get install -y --no-install-recommends \
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
        unzip \
        && apt-get autoremove -y && apt-get clean -y && rm -rf /var/lib/apt/lists/*
    
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
    RUN apt-get update && apt-get install -y --no-install-recommends gpg-agent software-properties-common && add-apt-repository -y ppa:ondrej/php \
        && apt-get autoremove -y && apt-get clean -y && rm -rf /var/lib/apt/lists/*

    RUN apt-get update; apt-get install -y --no-install-recommends \
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
        git \
        && apt-get autoremove -y && apt-get clean -y && rm -rf /var/lib/apt/lists/*
    RUN cp "/usr/lib/$(gcc -dumpmachine)"/libfuzzy.* /usr/lib; pecl install ssdeep && pecl install rdkafka && pecl install simdjson
    RUN git clone --recursive --depth=1 https://github.com/kjdev/php-ext-brotli.git && \
        cd php-ext-brotli && phpize && ./configure && make && make install

FROM "${DOCKER_HUB_PROXY}debian:bullseye-slim" as python-build
    ENV DEBIAN_FRONTEND noninteractive

    RUN apt-get update; apt-get install -y --no-install-recommends python3-pip git && apt-get autoremove -y && apt-get clean -y && rm -rf /var/lib/apt/lists/*
    RUN mkdir /wheels
    ARG MISP_TAG
    ARG MISP_COMMIT
    # Download MISP using git in the /var/www/ directory. Remove unnecessary items.
    RUN <<-EOF
            if [ ! -z "${MISP_COMMIT}" ]; then
                git clone https://github.com/MISP/MISP.git /var/www/MISP && cd /var/www/MISP && git checkout "${MISP_COMMIT}"
            else
                git clone --branch "${MISP_TAG}" --depth 1 https://github.com/MISP/MISP.git /var/www/MISP
            fi

            cd /var/www/MISP || exit; git -c submodule."PyMISP".update=none submodule update --init --recursive .

            # Remove modules only used during MISP build
            set -- 'coveralls' 'codecov$' 'requests-mock' 'nose'
            for mod in "$@"; do
                sed "/{$mod}/d" -i /var/www/MISP/requirements.txt
            done;

            # Add additional dependencies (container specific)
            set -- 'redis' 'python-magic' 'lief>=0.13.1' 'pydeep2' 'misp-lib-stix2' 'maec' 'mixbox'
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

    # Grab module wheels we need
    RUN pip3 wheel --no-cache-dir -w /wheels/ plyara pyzmq redis python-magic lief>=0.13.1 pymisp pydeep2 misp-lib-stix2 stix cybox maec mixbox

FROM "${DOCKER_HUB_PROXY}debian:bullseye-slim"

    ENV DEBIAN_FRONTEND noninteractive
    ARG MISP_TAG
    ARG MISP_COMMIT
    ARG PHP_VER
    RUN apt-get update; apt-get install -y --no-install-recommends \
        procps \
        sudo \
        nginx \
        supervisor \
        git \
        cron \
        openssl \
        gpg \
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
        curl jq \
        && apt-get autoremove -y && apt-get clean -y && rm -rf /var/lib/apt/lists/*

    # TODO: Copy misp from python-builder

    # Python Modules
    COPY --from=python-build /wheels /wheels
    RUN pip3 install --no-cache-dir /wheels/*.whl && rm -rf /wheels

    # PHP: install prebuilt libraries, then install the app's PHP deps
    COPY --from=php-build ["/usr/lib/php/${PHP_VER}/ssdeep.so", "/usr/lib/php/${PHP_VER}/rdkafka.so", "/usr/lib/php/${PHP_VER}/brotli.so", "/usr/lib/php/${PHP_VER}/simdjson.so", "/usr/lib/php/${PHP_VER}/"]

    # Do an early chown to limit image size
    COPY --from=python-build --chown=www-data:www-data /var/www/MISP /var/www/MISP
    COPY --from=composer-build --chown=www-data:www-data /tmp/Vendor /var/www/MISP/app/Vendor
    COPY --from=composer-build --chown=www-data:www-data /tmp/Plugin /var/www/MISP/app/Plugin

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
    RUN cp -R /var/www/MISP/app/files /var/www/MISP/app/files.dist; \
        cp -R /var/www/MISP/app/Config /var/www/MISP/app/Config.dist;

    # The spirit of the upstrem dockerization is to make:
    #   1) User and group aligned in terms of permissions
    #   2) Files executable and read only, because of some rogue scripts like 'cake'
    #   3) Directories writable, because sometimes MISP add new new files
    RUN find /var/www/MISP \( ! -user www-data -or ! -group www-data \) -exec chown www-data:www-data '{}' +; \
        find /var/www/MISP -not -perm 550 -type f -exec chmod 0550 '{}' +; \
        find /var/www/MISP -not -perm 770 -type d -exec chmod 0770 '{}' +;

    # Copy all our image specific files to appropriate locations
    COPY files/ /
    ENTRYPOINT [ "/entrypoint.sh" ]

    # Change Workdirectory
    WORKDIR /var/www/MISP
