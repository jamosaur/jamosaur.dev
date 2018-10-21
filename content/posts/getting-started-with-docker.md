---
title: "Getting Started With Docker"
date: 2018-10-21T13:42:00+01:00
draft: false
slug: getting-started-with-docker
tags: ['docker', 'php', 'laravel']
---

One of the most important things while working in a team is making sure that everyone runs the same stack, but also making sure it's as close to production as possible.

Developing with Laravel is great and gives you a few options out of the box (Homestead & Valet), but there are other tools you can use. Valet just configures nginx to run on your machine and provides an easy CLI to manage the sites you're developing on. Homestead is a pre-configured Vagrant configuration that includes a lot of different packages for you to use. Both options are great for begginers; but, in my opinion, they aren't great for managing environments across teams or mimicing production environments.

### Why Docker?

- Docker runs on macOS, Linux, and Windows.
- It ensures that all team members have the same reproducible environment.
- Set-up time is minimal for new hires.
- Doesn't require any of your stack to be installed locally.

### Getting Started

We're going to be using [Docker Compose](https://docs.docker.com/compose/), this is a tool for defining and running multi-container applications. Why multi-container? Each of our services should be independent! In the real world with scaling, you don't have multiple large boxes all running the same things. For example, your database may be an AWS RDS instance, which is completely seperate from your nginx/apache instances.

For our example application, we're going to be using nginx, php-fpm, mysql and mailhog.

We need to start by creating our compose file, so in your project root, create a new file called `docker-compose.yml`. In here, we're going to define our containers. My compose file has 5 services, `mysql`, `php-fpm`, `webserver`, `artisan`, `queue-worker`.

    version: "3.1"
    
    services: 
      
        mysql:
            image: mysql:5.7
            container_name: myapp-mysql
            working_dir: /application
            ports:
                - "3306:3306"
            environment:
                - MYSQL_ROOT_PASSWORD=password
                - MYSQL_DATABASE=myapp
                - MYSQL_USER=myuser
                - MYSQL_PASSWORD=password
        
        webserver:
            build:
                context: .
                dockerfile: docker/nginx/Dockerfile
            container_name: myapp-webserver
            hostname: api.myapp.test
            working_dir: /application
            volumes:
                - .:/application:cached
                - ./docker/nginx/nginx.conf:/etc/nginx/conf.d/default.conf
            ports:
                - "80:80"
                - "443:443"
            links:
                - php-fpm
        
        php-fpm:
            build:
                context: .
                dockerfile: docker/php-fpm/Dockerfile
            container_name: myapp-php-fpm
            working_dir: /application
            volumes:
                - .:/application:cached
                - ./docker/php-fpm/php-ini-overrides.ini:/etc/php/7.2/fpm/conf.d/99-overrides.ini
            links:
                - mysql
        
        artisan:
            build:
                context: .
                dockerfile: docker/artisan/Dockerfile
            container_name: myapp-artisan
            working_dir: /application
            volumes:
                - .:/application:cached
            links:
                - mysql
        
        queue-worker:
            build:
                context: .
                dockerfile: docker/artisan/Dockerfile
            container_name: enterprise-worker
            working_dir: /application
            volumes:
                - .:/application
            entrypoint: php artisan queue:work --tries=3
            links:
                - mysql

As you can see here, there's a couple of different things going on in some of the containers. some of them have `image` and others have `build`! Image just means that we are using a pre-built docker image that's been published to [docker hub](https://hub.docker.com/). `build` means that we're taking a local docker file and building it ourselves.

One of the important things in here is `links`, this enables containers to talk to each other.

### php-fpm container

From your application root directory, create a folder called `docker`, then one inside of it called `php-fpm`. 

`mkdir -p docker/php-fpm`

Inside of here, create a `Dockerfile`, mine looks like this:

    FROM php:7.2-fpm

    RUN apt-get update \
        && apt-get install -y gnupg libcurl4-openssl-dev sudo git libxslt-dev zlib1g-dev graphviz zip libmcrypt-dev \
          libicu-dev g++ libpcre3-dev libgd-dev libfreetype6-dev sqlite curl build-essential unzip gcc make autoconf \
          libc-dev pkg-config ruby git zlib1g-dev libfreetype6-dev libjpeg62-turbo-dev libpng-dev libxpm-dev libvpx-dev \
          netcat libmagickwand-dev libxml2-dev \
        && apt-get clean \
        && docker-php-ext-configure gd \
                --with-freetype-dir=/usr/lib/x86_64-linux-gnu/ \
                --with-jpeg-dir=/usr/lib/x86_64-linux-gnu/ \
                --with-xpm-dir=/usr/lib/x86_64-linux-gnu/ \
                --with-vpx-dir=/usr/lib/x86_64-linux-gnu/ \
        && docker-php-ext-install zip mbstring gettext curl pdo_mysql json intl gd soap \
        && pecl install imagick xdebug \
        && docker-php-ext-enable imagick xdebug
    
    WORKDIR "/application"

This means we're using the php image (tagged with `7.2-fpm`) as a base, then running commands like you would in your terminal. Here we're just installing some extensions such as `gd`, `soap`, `pdo_mysql` etc.

Take a second look at the compose file and notice that there is two items in `volumes`. One mounts our local application into `/application` and uses caching to speed things up. The second one takes `./docker/php-fpm/php-ini-overrides.ini` and places it in `/etc/php/7.2/fpm/conf.d/99-overrides.ini`. This allows us to override default settings in the `php-fpm` container. If you're happy with the defaults, remove this line from the compose file.

### Artisan container

The artisan dockerfile looks like this:

    FROM vcarreira/php7:latest
    
    ENTRYPOINT ["php", "artisan"]
    CMD ["list"]
    
_this is currently a WIP and needs to be refactored to use official PHP images!_

This image is used by the artisan container and also the queue worker. The queue worker has a different `entrypoint` in the `docker-compose.yml`, this makes the container automatically run `php artisan queue:work --tries=3` instead of `php artisan list`.

### nginx container

The dockerfile for the nginx container is this:

    FROM nginx:alpine

    RUN apk update && apk upgrade && \
        apk add --no-cache bash git openssh

It just uses the default nginx image, but we add in some other things to make our debugging easier. 

As per the `docker-compose.yml`, we have something in the volumes, just like the `php-fpm` container. This creates our `nginx.conf` file. Here is a very basic `nginx.conf` you can use.

    server {
        listen 80 default_server;
        server_name api.myapp.test;
        root /application/public;
        index index.php;
        if (!-e $request_filename) {
            rewrite ^.*$ /index.php last;
        }
    
        location ~ \.php$ {
            fastcgi_pass php-fpm:9000;
            fastcgi_index index.php;
            fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
            fastcgi_param PHP_VALUE "error_log=/var/log/nginx/application_php_errors.log";
            fastcgi_buffers 16 16k;
            fastcgi_buffer_size 32k;
            include fastcgi_params;
        }
    }

You can see in here that for the `fastcgi_pass`, we have `php-fpm:9000`. The docker container will resolve this to an internal IP that is given to your `php-fpm` container when it is created.

### Running compose

The final step to all of this is running it. All we need to do is build the containers that aren't pre-made and pull the pre-made ones!.

`docker-compose pull && docker-compose build`

This will take a few minutes as it downloads the containers and then builds your custom ones. While this is running, set up dnsmasq (or edit your hosts file) to make sure your local test domain is pointing to `127.0.0.1`.

Once `pull` and `build` have finished running, it's time to bring up the containers!

`docker-compose up` will bring them all up and show logs/output from each one in the terminal window that you run it from.

`docker-compose up -d` will bring them up, but background the process. You can use `docker-compose logs` to view the logs from the containers.

Open up your browser and go to `api.myapp.test` and you'll be presented with your application.