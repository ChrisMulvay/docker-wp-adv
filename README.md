# Docker WordPress Setup for Local Development

## Introduction

All the information found within this document come from a great article by titled:
**Build a solid WordPress dev environment with Docker**
**URL**: <https://aschmelyun.com/blog/build-a-solid-wordpress-dev-environment-with-docker/>

## Create a directory structure

    my-website-folder
        nginx/
            default.conf
        wordpress/
            (WordPress Source Files)
        docker-compose.yml
        nginx.dockerfile
        php.dockerfile

**Note**: _Downloand and install WordPress into the *wordpress/* directory in the above structure._

## Creating Our Stack

"A great rule of thumb when using Docker is that each container should provide a signle service, or handle a single process."

We will be creating a **LEMP** stack

- **L** (Linux)
- **E** (Nginx)
- **M** (MySQL)
- **P** (Php)

All we need to do is define the services (containers) we need at runtime.  
Docker provisions each one and wraps them all in a virtual umbrella network.
This means that each service will be accessible from every other container through the use of a hostname.

## Let's Get Started

Open the _docker-compose.yml_ file and add the following lines:

    version: '3.9'

    services:


- Version: 3.9, the newest version of the Docker Compose engine, not super useful but opens us up to newer syntax goodies.
- Services: Where we’ll specify the images that’ll make up our stack.

### Add Nginx

    nginx:
        build:
            context: .
            dockerfile: nginx.dockerfile
        ports: 
          - 80:80
        volumes: 
          - ./wordpress:/var/www/html
        depends_on: 
          - php 
          - mysql


#### Edit the *nginx.dockerfile*

    FROM nginx:stable-alpine

    AND ./nginx/default.conf /etc/nginx/conf.d/default.conf


#### Edit the *nginx/default.conf*

    upstream php {
        server unix:/tmp/php-cgi.socket;
        server php:9000;
    }

    server {
        listen 80;
        server_name wordpress-docker.test;

        root /var/www/html;

        index index.php;

        location / {
            try_files $uri $uri/ /index.php?$args;
        }

        location ~ \.php$ {
            include fastcgi.conf;
            fastcgi_intercept_errors on;
            fastcgi_pass php;
        }

        location ~* \.(js|css|png|jpg|jpeg|gif|ico)$ {
            expires max;
            log_not_found off;
        }  
    }


### Add MySQL

In our docker-compose.yml file add the following:

    mysql:
        image: mysql:latest
        restart: always
        ports: 
          - 3306:3306
        environment:
            MYSQL_DATABASE: wp
            MYSQL_USER: wp
            MYSQL_PASSWORD: secret
            MYSQL_ROOT_PASSWORD: secret

### Add PHP

Add the following to the bottom of our docker-compose.yml file:

    php:
        build:
            context: .
            dockerfile: php.dockerfile
        volumes: - ./wordpress:/var/www/html

#### php.dockerfile

    FROM php:7.4-fpm-alpine

    ADD ./php/www.conf /usr/local/etc/php-fpm.d/www.conf

    RUN addgroup -g 1000 wp && adduser -G wp -g wp -s /bin/sh -D wp

    RUN mkdir -p /var/www/html

    RUN chown wp:wp /var/www/html

    WORKDIR /var/www/html

    RUN docker-php-ext-install mysqli pdo pdo_mysql && docker-php-ext-enable pdo_mysql

#### Starting Docker

    docker-compose up --build

### Adding WP-CLI to docker-compose.yml

    wp:
        build:
            context: .
            dockerfile: php.dockerfile
        entrypoint: ['wp', '--allow-root']
        volumes: 
          - ./wordpress:/var/www/html