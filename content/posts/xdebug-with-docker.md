---
date: 2018-10-20T20:33:12+01:00
draft: false
title: Using Xdebug with docker and PhpStorm
slug: xdebug-with-docker-phpstorm
tags: ["php", "xdebug", "docker", "laravel"]
---

Over the last two weeks i have been working on a tricky bug, related to timezones and recurring bookings. After a while you get sick of making changes to your code, adding some `dd()`'s in there and re-running your request. Don't get me wrong, `dd()` is useful, sometimes quicker to use too. But if you're in a loop somewhere and want to find out what's happening, it's not really much help at all.

We're using docker (+ compose) to develop locally, and from what i've read it's a bit of a nightmare to set things up. I asked for some help to get this set up and [Leo](https://twitter.com/Phroggyy) pointed me to a [tutorial he wrote](https://blog.leosjoberg.com/post/xdebug-on-docker/) back in January. Parts of it worked for my set-up, but not all of it.

### Nginx

So to begin with, you need to make sure that nginx is set up properly. nginx needs to be able to respond with the correct `SERVER_NAME`.

If you don't have this set up already, add it in to your `nginx.conf`. For this tutorial i will be using `api.mypthub.test`, which is what we use, replace any instance of this in the tutorial with the domain you use.

Add it into your main server block and also into the PHP block.
    
    server {
      #
      server_name api.mypthub.test
      #
      
      location ~ \.php$ {
        #
        fastcgi_param SERVER_NAME api.mypthub.test;
        #
      }
    }

### PHP/Docker

Next up is changing our `php-fpm` container.

We need to make sure that Xdebug is installed, and that it gets set up correctly. Update your containers dockerfile to include the following:

    RUN pecl install xdebug-2.5.5 \
    && docker-php-ext-enable xdebug \
    && echo "xdebug.remote_enable=on\nxdebug.remote_autostart=1\nxdebug.remote_host=host.docker.internal\nxdebug.remote_port=9001\nxdebug.idekey=PHPSTORM" > /usr/local/etc/php/conf.d/xdebug.ini

After this has run, on our container the `xdebug.ini` will look like this:

    xdebug.remote_enable=on
    xdebug.remote_autostart=1
    xdebug.remote_host=host.docker.internal
    xdebug.remote_port=9001
    xdebug.idekey=PHPSTORM
    
We set the host to `host.docker.internal` here as it works for providing the host machines IP on every platform (thanks to [Patrick](https://www.pleckey.me/) for the tip!).

### PhpStorm

Last but not least, we have to set up PhpStorm so that it connects to Xdebug.

1. Open up preferences `cmd+,`
2. Navigate to: `Languages & Frameworks > PHP > Debug > DBGp Proxy`
3. Fill out the details 
  1. IDE Key: `PHPSTORM`
  2. Host: `127.0.0.1`
  3. Port: `9001`
4. Navigate to `Languages & Frameworks > PHP > Debug`
5. Set Xdebug port to `9001`
6. Navigate to `Languages & Frameworks > PHP > Servers`
7. Add a server named `api.mypthub.test` then add a path mapping to your project folder. Mine is `/application/api`, because my `docker-compose.yml` mounts a volume like this: `.:/application:cached`. Change this based on how your data is mounted in your container.

### Success!

Simply click on the `Run` menu in PhpStorm, then `Start listening for PHP debug connections`. Set some breakpoints in your code and send a request.

Use the debug panel to work things out instead of using `dd()` everywhere.

Good luck fixing those bugs ;)
