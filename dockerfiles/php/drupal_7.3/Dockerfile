FROM php:7.3.12-apache-buster

ADD addons/ /var/www/addons/
WORKDIR /var/www/html/
# CMD [ "php", "./index.php" ]

RUN apt-get update && apt-get install -y \
        libfreetype6-dev \
        libjpeg62-turbo-dev \
        libssl-dev \
        libcurl4-openssl-dev \
        pkg-config \
        libmcrypt-dev \
        libpng-dev \
        default-mysql-client \
        libxml2-dev \
        libzip-dev \
        libwebp-dev \
        libpng-dev \
        libxpm-dev \
        libzip-dev \
&& docker-php-ext-install -j$(nproc) iconv mbstring zip curl json mysqli pdo_mysql simplexml xml \
&& docker-php-ext-configure gd --with-gd --with-freetype-dir=/usr/include/ --with-jpeg-dir=/usr/include/ --with-webp-dir \
&& docker-php-ext-install -j$(nproc) gd \
&& a2enmod proxy \
&& a2enmod proxy_http \
&& a2enmod proxy_balancer \
&& a2enmod lbmethod_byrequests \
&& a2enmod proxy_ftp \
&& a2enmod proxy_connect \
&& a2enmod proxy_ajp \
&& a2enmod proxy_wstunnel \
&& a2enmod headers \
&& a2enmod rewrite \
&& a2enmod ssl \
&& mv /var/www/addons/composer.phar /usr/local/bin/composer \
&& chmod 777 /usr/local/bin/composer \
&& composer global require drush/drush:8.* \
&& ln -s /root/.composer/vendor/bin/drush /usr/local/bin/drush \
&& openssl req -new -newkey rsa:2048 -nodes -out localhost.csr -keyout localhost.key -subj "/C=CA/ST=Quebec/L=Montreal/O=myself/OU=Montreal/CN=localhost" \
&& openssl x509 -req -sha256 -days 365 -in localhost.csr -signkey localhost.key -out localhost.crt \
&& mv localhost.* /root \
&& service apache2 restart
