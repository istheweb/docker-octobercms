FROM php:5.6-apache

RUN apt-get update && apt-get install -y git-core \
  libjpeg-dev libmcrypt-dev libpng12-dev libpq-dev libsqlite3-dev \
  && rm -rf /var/lib/apt/lists/* \
  && docker-php-ext-configure gd --with-png-dir=/usr --with-jpeg-dir=/usr \
  && docker-php-ext-install gd mcrypt mysqli opcache pdo pdo_pgsql pdo_mysql pdo_sqlite zip

# recommended opcache PHP.ini settings
# see https://secure.php.net/manual/en/opcache.installation.php
RUN { \
		echo 'opcache.memory_consumption=128'; \
		echo 'opcache.interned_strings_buffer=8'; \
		echo 'opcache.max_accelerated_files=4000'; \
		echo 'opcache.revalidate_freq=2'; \
		echo 'opcache.fast_shutdown=1'; \
		echo 'opcache.enable_cli=1'; \
	} > /usr/local/etc/php/conf.d/opcache-recommended.ini

RUN curl -sS https://getcomposer.org/installer | php -- --install-dir=/usr/local/bin --filename=composer \
  && /usr/local/bin/composer global require hirak/prestissimo

RUN a2enmod rewrite

ENV OCTOBERCMS_TAG v1.0.419
ENV OCTOBERCMS_CHECKSUM 967d673cbc6977e823270127650d0e63c9164b0f
ENV OCTOBERCMS_CORE_BUILD 419
ENV OCTOBERCMS_CORE_HASH 386846ebe3802c4f192625743abb3f06

# Consolidated October CMS setup
RUN curl -o octobercms.tar.gz -fSL https://codeload.github.com/octobercms/october/legacy.tar.gz/{$OCTOBERCMS_TAG} \
  && echo "$OCTOBERCMS_CHECKSUM *octobercms.tar.gz" | sha1sum -c - \
  && tar -xzf octobercms.tar.gz \
  && OCTOBER_TMP_FOLDER=$(ls -d octobercms*/) \
  && find $OCTOBER_TMP_FOLDER -mindepth 1 -maxdepth 1 -exec mv -t . {} \; \
  && rmdir $OCTOBER_TMP_FOLDER \
  && rm octobercms.tar.gz \
  # match build dependencies
  && echo "Update composer.json: Require dependencies equal to or less than core build" \
  && sed -i.orig "s/\(\"october\/\([rain|system|backend|cms]*\)\": \"\(~1.0\)\"\)/\"october\/\2\": \"<=${OCTOBERCMS_TAG#v}\"/g" composer.json \
  && egrep -o "['\"]october\/[rain|system|backend|cms]*['\"]\s*:\s*['\"](.+?)['\"]" composer.json \
  && composer install --no-interaction --prefer-dist --no-scripts \
  # setup docker env
  && echo 'APP_ENV=docker' > .env \
  && mkdir config/docker \
  && echo "<?php return ['edgeUpdates' => false, 'disableCoreUpdates' => true];" > config/docker/cms.php \
  && echo "<?php return ['default' => 'sqlite'];" > config/docker/database.php \
  && echo "<?php return ['driver' => 'log'];" > config/docker/mail.php \
  # setup database
  && touch storage/database.sqlite \
  && chmod 666 storage/database.sqlite \
  && php artisan october:up \
  # set build params
  && php -r "use System\\Models\\Parameter; \
    require __DIR__.'/bootstrap/autoload.php'; \
    \$app = require_once __DIR__.'/bootstrap/app.php'; \
    \$app->make('Illuminate\\Contracts\\Console\\Kernel')->bootstrap(); \
    Parameter::set(['system::core.build'=>getenv('OCTOBERCMS_CORE_BUILD'), 'system::core.hash'=>getenv('OCTOBERCMS_CORE_HASH')]); \
    echo \"October CMS \\n Build: \",Parameter::get('system::core.build'), \"\\n Hash: \", Parameter::get('system::core.hash'), \"\\n\";" \
  # permissions
  && chown -R www-data:www-data /var/www/html

CMD ["apache2-foreground"]
