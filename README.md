# Craft CMS Docker Image

Lightweight Craft CMS 3 Image

Comes with Craft CMS 3 and support for PostgreSQL (`benkrll/craftcms:postgresql`) or MySQL (`benkrll/craftcms:mysql`).

Bring your own webserver and database.

## Features

- [Plugins and other dependencies](#plugins)  
  Automatically requires and removes additional plugins via DEPENDENCIES environment variable
- [Import SQL files](#import-database)  
  Automatically restores database backups located under ./backups (See volumes in docker-compose.yml below). To create a database backup use the built-in backup tool within the Craft CMS Control Panel.
- Completely non-interactive installation of Craft CMS and plugins
- [redis](#redis)
- imagemagick
- If you want to use PostgreSQL use the `benkrll/craftcms:postgresql` image
- If you want to use MySQL use the `benkrll/craftcms:mysql` image

## Getting started

You only need two files:

- docker-compose.yml
- default.conf

Add `backups/.ignore` to your .gitignore file. This file is used for automatic db restores. Files listed in `backups/.ignore` do not get imported on start up.

### docker-compose

#### PostgreSQL Example

```yml
# docker-compose.yml
version: '2.1'

services:
  nginx:
    image: nginx:alpine
    ports:
      - 80:80
    depends_on:
      - craft
    volumes_from:
      - craft
    volumes:
      - ./default.conf:/etc/nginx/conf.d/default.conf # nginx configuration (see below)
      - ./assets:/var/www/html/web/assets # For static assets (media, js and css).

  craft:
    image: benkrll/craftcms:postgresql
    depends_on:
      - postgres
    volumes:
      - ./assets:/var/www/html/web/assets:z
      - ./backups:/var/www/html/storage/backups # Used for db restore on start.
      - ./templates:/var/www/html/templates # Craft CMS template files
      - ./translations:/var/www/html/translations
      - ./redactor:/var/www/html/config/redactor
    environment:
      DEPENDENCIES: >- # additional composer packages
        yiisoft/yii2-redis
        craftcms/redactor:2.0.1

      CRAFTCMS_EMAIL: admin@company.com
      CRAFTCMS_USERNAME: admin
      CRAFTCMS_PASSWORD: super-secret-password
      CRAFTCMS_SITENAME: Craft CMS Installation
      CRAFTCMS_SITEURL: http://dev.project.com # Optional
      CRAFTCMS_LANGUAGE: de-AT # Optional

      AUTO_UPDATE: 'false' # Enable/disable auto updates for all composer packages (including Craft CMS, Default: true)

      REDIS_HOST: redis
      SESSION_DRIVER: redis
      CACHE_DRIVER: redis

      DB_DSN: pgsql:host=postgres;dbname=craft
      DB_SERVER: postgres
      DB_NAME: craft
      DB_USER: craft
      DB_PASSWORD: secret
      DB_DATABASE: craft
      DB_SCHEMA: public
      DB_DRIVER: pgsql
      DB_PORT: 5432
      DB_TABLE_PREFIX: ut

  postgres:
    image: postgres:10.3-alpine
    environment:
      POSTGRES_ROOT_PASSWORD: root
      POSTGRES_USER: craft
      POSTGRES_PASSWORD: secret
      POSTGRES_DB: craft
    volumes:
      # Persistent data
      - pgdata:/var/lib/postgresql/data

  redis:
    image: redis:4-alpine
    volumes:
      - redisdata:/data

volumes:
  pgdata:
  redisdata:
```

#### MySQL Database Example

```yml
# docker-compose.yml
version: '2.1'

services:
  nginx:
    image: nginx:alpine
    ports:
      - 80:80
    depends_on:
      - craft
    volumes_from:
      - craft
    volumes:
      - ./default.conf:/etc/nginx/conf.d/default.conf # nginx configuration (see below)
      - ./assets:/var/www/html/web/assets # For static assets (media, js and css).

  craft:
    image: benkrll/craftcms:mysql # Use mysql instead of postgresql
    depends_on:
      - mariadb
    volumes:
      - ./assets:/var/www/html/web/assets:z
      - ./backups:/var/www/html/storage/backups # Used for db restore on start.
      - ./templates:/var/www/html/templates # Craft CMS template files
      - ./translations:/var/www/html/translations
      - ./redactor:/var/www/html/config/redactor
    environment:
      DEPENDENCIES: >- # additional composer packages
        yiisoft/yii2-redis
        craftcms/redactor:2.0.1

      CRAFTCMS_EMAIL: admin@company.com
      CRAFTCMS_USERNAME: admin
      CRAFTCMS_PASSWORD: super-secret-password
      CRAFTCMS_SITENAME: Craft CMS Installation
      CRAFTCMS_SITEURL: http://dev.project.com # Optional

      AUTO_UPDATE: 'false' # Enable/disable auto updates for all composer packages (including Craft CMS, Default: true)

      REDIS_HOST: redis
      SESSION_DRIVER: redis
      CACHE_DRIVER: redis

      DB_DSN: mysql:host=mariadb;dbname=craft
      DB_SERVER: mariadb
      DB_NAME: craft
      DB_USER: craft
      DB_PASSWORD: secret
      DB_DATABASE: craft
      DB_SCHEMA: public
      DB_DRIVER: mysql
      DB_PORT: 3306
      DB_TABLE_PREFIX: ut

  mariadb:
    image: mariadb:10.1
    environment:
      MYSQL_ROOT_PASSWORD: root
      MYSQL_USER: craft
      MYSQL_PASSWORD: secret
      MYSQL_DATABASE: craft
    volumes:
      # Persistent data
      - dbdata:/var/lib/mysql

  redis:
    image: redis:4-alpine
    volumes:
      - redisdata:/data

volumes:
  dbdata:
  redisdata:
```

### nginx configuration

Create a file called **default.conf** in to your project directory:

```nginx
# default.conf

server {
    listen 80 default_server;
    listen [::]:80 default_server;
    server_name localhost;

    index index.php index.html;
    error_log  /var/log/nginx/error.log;
    access_log /var/log/nginx/access.log;
    root /var/www/html/web;
    charset utf-8;

    # Root directory location handler
    location / {
        try_files $uri/index.html $uri $uri/ /index.php?$query_string;
    }

    location ~ \.php$ {
        try_files $uri $uri/ /index.php?$query_string;
        fastcgi_split_path_info ^(.+\.php)(/.+)$;
        fastcgi_pass craft:9000;
        fastcgi_index index.php;
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_param PATH_INFO $fastcgi_path_info;
    }
}
```

## Plugins

Add your plugins to the environment variable DEPENDENCIES. A script then automatically adds or removes those dependencies when you create the container.

In a docker-compose file:

```yaml
volumes:
  - ./path/to/plugin:/plugins/plugin # If you want to use a local plugin with composer

environment:
  DEPENDENCIES: >- # additional composer packages
    craftcms/redactor:2.0.1
    [vendor/package-name:branch-name]https://url-to-the-git-repo.git # branch-name needs to be prefixed with 'dev-'. (e.g. dev-development for branch development)
    [vendor/package-name:version]/path/to/volume # Version as 1.0.0
```

You can add plugins from a public Git Repo, or from a local folder on your machine as Well.
For local Plugins take care that you added a volume so that docker can use the plugin.

If you change your dependencies, run `docker-compose down && docker-compose up` to remove and recreate your container.

## Import database

Create another volume which gets mounted to /var/www/html/storage/backups

```
    volumes:
      - ./backups:/var/www/html/storage/backups
```

When you create a db dump from within your Craft CMS CP, a sql file gets automatically saved in this folder and you can add it to your git repository.

On startup the docker image looks for sql or zip files in this folder and imports them, unless its filename is listed in a special file called `.ignore`. This prevents importing the same file twice.

After successfully importing the db dump, this file gets added to `.ignore`, so you don't have to worry about that.

## Redis

Create `config/app.php` and add the following lines:

```php
return [
    'components' => [
        'cache' => [
            'class' => yii\redis\Cache::class,
            'defaultDuration' => 86400,
            'redis' => [
                'hostname' => getenv('REDIS_HOST'),
                'port' => 6379,
                'database' => 1,
            ],
        ],
    ],
];
```

Add `yiisoft/yii2-redis` to your DEPENDENCIES env variable and create another volume for your redis configuration

```yaml
volumes:
  - ./config/app.php:/var/www/html/config/app.php
```

Add a redis service to your docker-compose.yml

```yaml
redis:
  image: redis:4-alpine
  volumes:
    - redisdata:/data
```

## Finish setup

Run `docker-compose up` and visit http://localhost/admin. Voilà!

## Known Issues

- Docker volumes and file system permissions  
  See https://medium.com/@nielssj/docker-volumes-and-file-system-permissions-772c1aee23ca
