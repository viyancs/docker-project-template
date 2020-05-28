FROM php:7.2-fpm
ENV COMPOSER_VERSION=1.6.5 \
    NGINX_VERSION=1.16.1-1~buster \
    NJS_VERSION=1.16.1.0.3.7-1~buster
    

ADD https://raw.githubusercontent.com/mlocati/docker-php-extension-installer/master/install-php-extensions /usr/local/bin/

# this is a sample BASE image, that php_fpm projects can start FROM
# it's got a lot in it, but it's designed to meet dev and prod needs in single image
# I've tried other things like splitting out php_fpm and nginx containers
# or multi-stage builds to keep it lean, but this is my current design for
## single image that does nginx and php_fpm
## usable with bind-mount and unique dev-only entrypoint file that builds
## some things on startup when developing locally
## stores all code in image with proper default builds for production
# install apt dependencies
# some of these are not needed in all php projects
# NOTE: you should prob use specific versions of some of these so you don't break your app
RUN apt-get update && apt-get install --no-install-recommends --no-install-suggests -y \
    apt-transport-https \
    ca-certificates \
    openssh-client \
    curl \ 
    dos2unix \
    git \
    gnupg2 \
    dirmngr \
    g++ \	
    jq \
    libedit-dev \
    libfcgi0ldbl \
    libfreetype6-dev \
    libicu-dev \
    libjpeg62-turbo-dev \
    libmcrypt-dev \
    libpq-dev \
    libxml2-dev \
    supervisor \
    unzip \
    wget \
    vim \
    zip \
    strace \
    && rm -r /var/lib/apt/lists/*

# install extension using the tools from community 
RUN chmod uga+x /usr/local/bin/install-php-extensions && sync && \
    install-php-extensions \ 
    xdebug \
    imagick \
    tokenizer \
    bcmath \
    ctype \
    sqlite3 \
    pdo_sqlite \
    pgsql \
    pdo_pgsql \
    openssl \
    mbstring \
    fileinfo 

# enable extension & manual configuration 
RUN echo "zend_extension=/usr/local/lib/php/extensions/no-debug-non-zts-20170718/xdebug.so" >> /usr/local/etc/php/php.ini    
RUN echo "error_reporting = E_ALL" >> /usr/local/etc/php/conf.d/docker-php-ext-xdebug.ini
RUN echo "display_startup_errors = On" >> /usr/local/etc/php/conf.d/docker-php-ext-xdebug.ini
RUN echo "display_errors = On" >> /usr/local/etc/php/conf.d/docker-php-ext-xdebug.ini
RUN echo "xdebug.idekey=\"PHPSTORM\"" >> /usr/local/etc/php/conf.d/docker-php-ext-xdebug.ini
RUN echo "xdebug.remote_port=9000" >> /usr/local/etc/php/conf.d/docker-php-ext-xdebug.ini
RUN echo "xdebug.remote_enable=1" >> /usr/local/etc/php/conf.d/docker-php-ext-xdebug.ini
RUN echo "extension=imagick.so" >> /usr/local/etc/php/php.ini

# Install extensions using the helper script provided by the base image
RUN docker-php-ext-install \
    pdo_mysql \
    mysqli \
    json \
    readline \
    gd \
    intl

# configure gd
RUN docker-php-ext-configure gd \
    --with-freetype-dir=/usr/include/freetype2 \
    --with-jpeg-dir=/usr/include/

# configure intl
RUN docker-php-ext-configure intl

# check mod install
RUN php -m

###########
# PHPUnit #
###########
RUN wget https://phar.phpunit.de/phpunit-7.4.phar
RUN chmod +x phpunit-7.4.phar
RUN mv phpunit-7.4.phar /usr/local/bin/phpunit
RUN phpunit --version

###################
# PHP Codesniffer #
###################
RUN curl -OL https://squizlabs.github.io/PHP_CodeSniffer/phpcs.phar
RUN mv phpcs.phar /usr/local/bin/phpcs
RUN chmod +x /usr/local/bin/phpcs

# install nginx (copied from official nginx Dockerfile)
RUN NGINX_GPGKEY=573BFD6B3D8FBC641079A6ABABF5BD827BD9BF62; \
	found=''; \
	for server in \
		ha.pool.sks-keyservers.net \
		hkp://keyserver.ubuntu.com:80 \
		hkp://p80.pool.sks-keyservers.net:80 \
		pgp.mit.edu \
	; do \
		echo "Fetching GPG key $NGINX_GPGKEY from $server"; \
		apt-key adv --keyserver "$server" --keyserver-options timeout=10 --recv-keys "$NGINX_GPGKEY" && found=yes && break; \
	done; \
	test -z "$found" && echo >&2 "error: failed to fetch GPG key $NGINX_GPGKEY" && exit 1; \
	echo "deb http://nginx.org/packages/debian/ buster nginx" >> /etc/apt/sources.list.d/nginx.list \
	# echo "deb-src http://nginx.org/packages/debian/ buster nginx" >> /etc/apt/nginx.list \
	&& apt-get update \
	&& apt-get install --no-install-recommends --no-install-suggests -y \
						nginx=${NGINX_VERSION} \
						nginx-module-xslt=${NGINX_VERSION} \
						nginx-module-geoip=${NGINX_VERSION} \
						nginx-module-image-filter=${NGINX_VERSION} \
						nginx-module-njs=${NJS_VERSION} \
						gettext-base \
	&& rm -rf /var/lib/apt/lists/*

# forward nginx request and error logs to docker log collector
RUN ln -sf /dev/stdout /var/log/nginx/access.log \
	&& ln -sf /dev/stderr /var/log/nginx/error.log

# install composer so we can run dump-autoload at entrypoint startup in dev
# copied from official composer Dockerfile
ENV PATH="/composer/vendor/bin:$PATH" \
    COMPOSER_ALLOW_SUPERUSER=1 \
    COMPOSER_VENDOR_DIR=/var/www/vendor \
    COMPOSER_HOME=/composer

RUN curl --silent --fail --location --retry 3 --output /tmp/installer.php --url https://raw.githubusercontent.com/composer/getcomposer.org/cb19f2aa3aeaa2006c0cd69a7ef011eb31463067/web/installer \
 && php -r " \
    \$signature = '48e3236262b34d30969dca3c37281b3b4bbe3221bda826ac6a9a62d6444cdb0dcd0615698a5cbe587c3f0fe57a54d8f5'; \
    \$hash = hash('sha384', file_get_contents('/tmp/installer.php')); \
    if (!hash_equals(\$signature, \$hash)) { \
      unlink('/tmp/installer.php'); \
      echo 'Integrity check failed, installer is either corrupt or worse.' . PHP_EOL; \
      exit(1); \
    }" \
 && php /tmp/installer.php --no-ansi --install-dir=/usr/bin --filename=composer --version=${COMPOSER_VERSION} \
 && composer --ansi --version --no-interaction \
 && rm -f /tmp/installer.php

COPY supervisord.conf /etc/supervisor/conf.d/supervisord.conf

CMD ["/usr/bin/supervisord", "-n", "-c", "/etc/supervisor/conf.d/supervisord.conf"]
