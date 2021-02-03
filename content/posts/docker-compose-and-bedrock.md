---
title: "Docker Compose + roots/bedrock"
date: 2020-11-01T23:12:03Z
draft: false
---

### A quick start guide

## Pre-Reqs

* Docker installed
* PHP Composer installed

## Steps

1. `composer create-project roots/bedrock my-project`
1. `cd my-project`
1. `ln -s web html` as Apache will serve /var/www/html by default, this saves us from having to change that
1. Create a `docker-compose.yaml` with the following:
    ```yaml
    version: '3.1'

    services:

      wordpress:
        image: php:7-apache
        restart: always
        ports:
          - 8080:80
        volumes:
          - .:/var/www

      db:
        image: mysql:5.7
        restart: always
        environment:
          MYSQL_DATABASE: database_name
          MYSQL_USER: database_user
          MYSQL_PASSWORD: database_password
          MYSQL_RANDOM_ROOT_PASSWORD: '1'
        volumes:
          - db:/var/lib/mysql

    volumes:
      db:
    ```
1. `vi .env` to change `DB_HOST='localhost'` to `DB_HOST='db'`
1. `docker-compose up`
1. Open http://localhost:8080


## Going Further

### Installing a theme:

1. `composer require wpackagist-theme/hestia`

More info: https://wpackagist.org, https://roots.io/docs/bedrock/master/composer/
