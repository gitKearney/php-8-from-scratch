# Compile PHP 8 on Ubuntu Server 20.04
Want to be on the bleeding edge, all the time? Me too! I prefer to compile PHP
from source because I get to have a binary that is small and I get to have the
latest version rather than waiting for the distro to release it

# TOC
 * [Ubuntu Packages](#packages)
 * [Add PHP8 To Your Path](#add-to-path)
 * [Download PHP 8](#download-php)
 * [Create A Build Script](#build)
 * [Compile PHP](#compile-php)
 * [PHP INI File](#php-ini)
 * [Download Composer](#php-composer)
 * [Configure PHP-FPM](#php-fpm)
 * [PHP Built-In Server](#php-s)
 * [PHP-FPM nGinx](#php-nginx)
 * [PHP-FPM Lighttpd](#php-lighttpd)
 * [XDebug](#xdebug)


# <a name="packages"></a>Download the necessary packages
This is tailored for Laravel, Symfony, Drupal - some of the these packages *probably* won't be necessary but, why not just get everything, eh?

    apt-get install build-essential pkg-config \
      libxml2-dev libssl-dev libsqlite3-dev libzip-dev \
      libcurl4-openssl-dev libonig-dev libsodium-dev \
      libargon2-dev zip libgd-dev libicu-dev autoconf

# <a name="add-to-path"></a>Add PHP to path
Add PHP 8 to our path. Edit your *.bashrc* file and add

    if [ -d "$HOME/bin/php8/bin" ] ; then
      PATH="$HOME/bin/php8/bin:$PATH"
    fi

# <a name="download-php"></a>Download PHP 8
Let's grab the latest PHP 8 version

    wget https://www.php.net/distributions/php-8.1.9.tar.gz -O php-8.1.9.tar.gz
    tar xf php-8.1.9.tar.gz
    
# <a name="build"></a>Create A Build Script
`cd php-8.1.8` and create the following file *build_php.sh*

    #!/bin/sh

    ######################
    # filename: build_php.sh
    #
    # STEP 1: sh build_php.sh
    # STEP 2: make -j number_of_cores_or_processors_CPU_has
    # STEP 3: make install
    ######################

    INSTALL_DIR=$HOME/bin/php8

    mkdir -p $INSTALL_DIR

    ./configure --prefix=$INSTALL_DIR \
      --enable-bcmath \
      --enable-fpm \
      --with-fpm-user=www-data \
      --with-fpm-group=www-data \
      --disable-cgi \
      --enable-mbstring \
      --enable-shmop \
      --enable-sockets \
      --enable-sysvmsg \
      --enable-sysvsem \
      --enable-sysvshm \
      --with-zlib \
      --with-curl \
      --without-pear \
      --with-openssl \
      --enable-pcntl \
      --with-password-argon2 \
      --with-sodium \
      --with-pear \
      --enable-intl \
      --with-external-gd

## If working With MariaDB or Postgres add these lines
If you are using MariaDB or PostgreSQL (or both) you will need to add some lines to the compile script as well as install the databases

## INSTALL DATABASES
First you have to install the databases

    apt-get install mariadb-server
    apt-get install postgresql postgresql-contrib libpq-dev

Then you need to add the following lines to the configure script

    # if working with MariaDB
    --with-pdo-mysql=mysqlnd

    # if working with Postgres
    --with-pdo-pgsql

# <a name="compile-php"></a>Compile PHP
Now that our build script it done (you should be in the php8.1.9 dir)

 * sh build_php.sh
 * make -j 2
 * make install

# <a name="php-ini"></a>Setup the INI file
Copy the file *php.ini-development* from the PHP 8.1.9 directory to our path

    # run from the php-8.1.9 directory
    cp php.ini-development ~/bin/php8/lib/php.ini
    
Make the following changes to the *php.ini* file

 * set the timezone: `date.timezone=America/Chicago` (see https://www.php.net/manual/en/timezones.php)
 * fix CGI: `cgi.fix_pathinfo=1`

# <a name="php-composer"></a>Download composer

    cd ~/bin
    php -r "copy('https://getcomposer.org/installer', 'composer-setup.php');"
    php composer-setup.php
    ln -s composer.phar composer
    unlink composer-setup.php
    

# <a name="php-fpm"></a>Configure PHP-FPM

## Create PHP-FPM config file

    cd ~/bin/php8/etc/; mv php-fpm.conf.default php-fpm.conf
    cd ~/bin/php8/etc/php-fpm.d/; mv www.conf.default www.conf

## Start PHP-FPM
You can start PHP-FPM service by running it

    sudo ~/bin/php8/sbin/php-fpm

# Webservers: nGinx, lighttpd, PHP -S
You have the option of serving your web app multiple ways.
nGinx and lighttpd are 2 good options. both are lightweight
and production ready. The 3rd option is running the PHP web-server `php -S`. This is a good option actually, but isn't
production ready

## <a name="php-s"></a>PHP -S
This is definitely the simplest and doesn't even require PHP-FPM

    cd /path/to/php/code
    php -S localhost:8080 -t /path/to/index.php

## <a name="php-nginx"></a>nGinx

    sudo apt-get install nginx

create a development config

    cd /etc/nginx/sites-available/
    sudo touch devel

Now, set devel to point to our development directory

    # filename: /etc/nginx/sites-available/devel
    server {
        listen 80 default_server;
        listen [::]:80 default_server;

        # path to code (index.php should be in this directory)
        root /path/to/php-code;

        index index.php;

        server_name phpdevbox;

        location / {
            try_files $uri $uri/ /index.php?query_string;
        }

        location ~ \.php$ {
            include snippets/fastcgi-php.conf;
            fastcgi_pass 127.0.0.1:9000;
        }

        location ~ /\.ht {
            # deny access to .htaccess files
            deny all;
        }
    }

Enable our new devel site

    cd /etc/nginx/sites-enabled
    unlink default
    ln -s /etc/nginx/sites-available/devel

    sudo systemctl reload nginx.service # reload nginx

## <a name="php-lighttpd"></a>lighttpd
This is a lightweight extremely fast webserver capable of serving 10,000 requests. With docker becoming more popular,
Lighttpd makes more sense than nginx (which performs better)
with multicores

    sudo apt-get install lighttpd
    sudo systemctl status lighttpd.service

Edit the file `/etc/lighttpd/conf-available/15-fastcgi-php.conf`

        fastcgi.server += (
            ".php" => ((
                "host" => "127.0.0.1",
                "port" => "9000",
                "broken-scriptfilename" => "enable"
            ))
        )

Edit the file `/etc/lighttpd/lighttpd.conf`

    # server.document-root = "/var/www/html"
    server.document-root = "/path/to/php"

Enable Fast CGI

    sudo lighttpd-enable-mod fastcgi
    sudo lighttpd-enable-mod fastcgi-php
    
 Enable rewrite to remove `index.php` from URL
 
    sudo lighttpd-enable-mod rewrite
    
 and edit the file `/etc/lighttpd/lighttpd.conf` and add the line
 
     url.rewrite-if-not-file = ( "" => "/index.php${url.path}${qsa}" )
    
Now, restart lighttpd

    sudo systemctl restart lighttpd.service

# <a name="xdebug"></a>Enabling XDebug
The PECL repo seems to always have the latest version of XDEBUG available, so you no longer
need to compile it and it install it manually. 

    pecl install xdebug

Open the ~/bin/php8/lib/php.ini file and add the lines

    [xdebug]
    zend_extension=xdebug
    xdebug.mode=develop,debug
    xdebug.client_host=IP.OF.YOUR.COMPUTER
    xdebug.start_with_request=yes

The client host is the IP of your computer running VS Code or PHPStorm
