# Craft CMS Docker Image

Lightweight Craft CMS 3 Image

Comes with Craft CMS 3 and support for PostgreSQL or MySQL (`urbantrout/craftcms:mysql`).

Bring your own webserver and database.

## Features

* Automatically requires and removes additional plugins via DEPENDENCIES environment variable
* Automatically restores database backups located under ./backups (See volumes in docker-compose.yml below). To create a database backup, just use the built-in backup tool within the Craft CMS Control Panel.
* pdo_pgsql
* pg_dump for backups
* redis
* imagemagick
* If you want to use MySQL instead of PostgreSQL just use the `urbantrout/craftcms:mysql` image

## Example Setup

You only need two files:

* docker-compose.yml
* default.conf

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
      - ./assets:/var/www/html/web/assets # For static assets (media, js and css). We don't need PHP for them.

  craft:
    image: urbantrout/craftcms
    depends_on:
      - postgres
    volumes:
      - ./backups:/var/www/html/storage/backups # Used for db restore on start.
      - ./templates:/var/www/html/templates # Craft CMS template files
      - ./translations:/var/www/html/translations
      - ./redactor:/var/www/html/config/redactor
    environment:
      DEPENDENCIES: >- # additional composer packages (must be comma separated)
        yiisoft/yii2-redis,
        craftcms/redactor,

      REDIS_HOST: redis
      SESSION_DRIVER: redis
      CACHE_DRIVER: redis

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
      - ./assets:/var/www/html/web/assets # For static assets (media, js and css). We don't need PHP for them.

  craft:
    image: urbantrout/craftcms:mysql # Use mysql instead of postgresql
    depends_on:
      - mariadb
    volumes:
      - ./backups:/var/www/html/storage/backups # Used for db restore on start.
      - ./templates:/var/www/html/templates # Craft CMS template files
      - ./translations:/var/www/html/translations
      - ./redactor:/var/www/html/config/redactor
      - /path/to/plugin:/plugins/plugin # If you want a local plugin for composer
    environment:
      DEPENDENCIES: >- # additional composer packages (must be comma separated)
        yiisoft/yii2-redis,
        craftcms/redactor,
        [vendor/package-name:branch-name]https://url-to-the-git-repo.git # Branch name should be prefixed with dev-${branchname}
        [vendor/package-name:version]/path/to/volume # Version as 1.0.0

      REDIS_HOST: redis
      SESSION_DRIVER: redis
      CACHE_DRIVER: redis

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

Just add your plugins to the environment variable DEPENDENCIES. A script then automatically adds or removes those dependencies when you create the container.

In a docker-compose file:

```yaml
    environment:
      DEPENDENCIES: >- # additional composer packages (must be comma separated)
        craftcms/redactor,
```

You can add plugins from a public Git Repo, or from a local folder on your machine as Well.
For local Plugins take care that you added a volume so that docker can use the plugin.

If you change your dependencies, just run `docker-compose down && docker-compose up` to remove and recreate your container.

## Finish setup

Run `docker-compose up` and visit http://localhost/admin. Voilà!

## Known Issues

`/run.sh: line 66: .ignore: Permssion denied`
On Linux you need to change the owner and group of the directory ./backups to 82:82, otherwise docker cannot write to the .ignore file. This also applies to any other directory which you mount as volume and docker should be able to write to (e.g. assets/media for Craft CMS Assets). 82 is the UID and GID of www-data inside the docker container.
