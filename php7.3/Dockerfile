FROM php:7.3.33-fpm-alpine3.15

RUN apk update && apk add --no-cache tzdata vim busybox git unzip wget \
    #gd依赖
    freetype-dev libjpeg-turbo-dev libpng-dev libwebp-dev \
    #soap xmlrpc依赖
    libxml2-dev tidyhtml-dev libxslt-dev \
    #zip依赖
    libzip-dev \
    #intl依赖
    icu-dev \
    #gettext依赖
    gettext-dev \
    #剥离符号表 \
    binutils \
    #编译依赖
    libmemcached-dev zlib-dev autoconf gcc make g++ zlib-dev linux-headers cmake openssl-dev curl-dev \
    #临时设置make编译线程数为核心数
    && export GNUMAKEFLAGS=-j$(nproc) \
    #安装swoole redis扩展
    && pecl install -D "enable-openssl='yes' enable-http2='yes'" swoole-4.6.7 \
    && pecl install redis-5.3.7 \
    && docker-php-ext-enable swoole redis \
    #安装热更新依赖
    && cd /tmp \
    && wget https://github.com/emcrisostomo/fswatch/releases/download/1.18.0/fswatch-1.18.0.tar.gz \
    && tar -xzvf fswatch-1.18.0.tar.gz \
    && cd fswatch-1.18.0 \
    && ./configure \
    && make \
    && make install \
    #清理临时文件
    && rm -rf /tmp/* \
    #安装常用php扩展
    && docker-php-ext-configure gd \
    --with-webp-dir=/usr/include \
    --with-jpeg-dir=/usr/include \
    --with-freetype-dir=/usr/include \
    && docker-php-ext-install \
    gd \
    bcmath \
    exif \
    gettext \
    sockets \
    mysqli \
    pcntl \
    pdo_mysql \
    shmop \
    sysvsem \
    soap \
    intl \
    xmlrpc \
    zip \
    opcache \
    #清理符号表，让镜像更小
    && find $(php-config --extension-dir) -type f -name "*.so" -exec strip {} + \
    && strip /usr/local/bin/fswatch \
    #删除编译后无用依赖
    && apk del binutils tidyhtml-dev libxslt-dev libmemcached-dev zlib-dev autoconf gcc make g++ linux-headers cmake openssl-dev curl-dev

#默认关闭cli模式下opcache
RUN echo "opcache.enable_cli=0" >> /usr/local/etc/php/conf.d/docker-php-ext-opcache.ini \
    && echo "opcache.interned_strings_buffer=128" >> /usr/local/etc/php/conf.d/docker-php-ext-opcache.ini \
    && echo "opcache.memory_consumption=1024" >> /usr/local/etc/php/conf.d/docker-php-ext-opcache.ini \
    && echo "opcache.max_accelerated_files=100000" >> /usr/local/etc/php/conf.d/docker-php-ext-opcache.ini \
    && echo "opcache.validate_timestamps=1" >> /usr/local/etc/php/conf.d/docker-php-ext-opcache.ini \
    && echo "opcache.revalidate_freq=60" >> /usr/local/etc/php/conf.d/docker-php-ext-opcache.ini

#安装composer
RUN php -r "copy('https://getcomposer.org/installer', 'composer-setup.php');" \
    && php composer-setup.php \
    && php -r "unlink('composer-setup.php');" \
    && mv composer.phar /usr/local/bin/composer  \
    && composer config -g repo.packagist composer https://mirrors.aliyun.com/composer/

#更改时区为东八区
RUN cp /usr/share/zoneinfo/Asia/Shanghai /etc/localtime \
    && echo "Asia/Shanghai" > /etc/timezone \
    && echo "date.timezone=Asia/Shanghai" > /usr/local/etc/php/conf.d/tzone.ini

#设置composer bin到环境变量
ENV PATH=$PATH:/root/.composer/vendor/bin

#软链接php到/bin下 避免crontab里需要写完整php路径 并生成crontab和command空文件
RUN ln -s /usr/local/bin/php /bin/php \
    && echo "" > /crontab \
    && echo "" > /command \
    && echo "" > /command.log

#php cli进程守护
RUN wget --no-check-certificate -O /bin/commandGuard https://cdn.nikm.cn/util/commandGuard-$(uname -m) \
    && chmod +x /bin/commandGuard

#启动cron定时任务和php-fpm
RUN echo "#!/bin/sh" > /start.sh \
    && echo "nohup commandGuard -cmd /command -log /command.log >> /command.log 2>&1 &" >> /start.sh \
    && echo "crontab /crontab" >> /start.sh \
    && echo "crond" >> /start.sh \
    && echo "php-fpm" >> /start.sh \
    && chmod +x /start.sh

CMD ["/start.sh"]