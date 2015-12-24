# Magento2 (Varnish + PHP7 + Redis) docker-compose infrastructure

## Infrastructure overview
* Container 1: MariaDB
* Container 2: Redis (we'll use it for Magento's cache)
* Container 3: Apache 2.4 + PHP 7 (modphp)
* Container 4: Cron
* Container 5: Varnish 4

###Why a separate cron container?
First of all containers should be (as far as possible) single process, but the most important thing is that (if someday we'll be able to deploy this infrastructure in production) we may need a cluster of apache+php containers but a single cron container running.

Plus, with this separation, in the context of a docker swarm, you may be able in the future to separare resources allocated to the cron container from the rest of the infrastructure.

## Downloading Magento2
```
composer create-project --repository-url=https://repo.magento.com/ magento/project-community-edition magento2
```

## Starting all docker containers
```
docker-compose up -d
```
The fist time you run this command it's gonna take some time to download all the required images from docker hub.

## Do you want sample data?
First we need to enter the apache+php container:
```
docker exec -it dockermagento2_apache_1 bash
```

Then execute (from within the container):
```
php bin/magento sampledata:deploy
composer update
```
If you see an error during the "composer update" then lauch it another time ;)

## Install Magento2

open your browser to the address:
```
http://magento2.docker/
```
and use the wizard to install Magento2.

## Deploy static files
```
docker exec -it dockermagento2_apache_1 bash
php magento dev:source-theme:deploy
php magento setup:static-content:deploy
```

## Enable Redis for Magento's cache
open magento2/app/etc/env.php and add these lines:
```php
'cache' => array(
  'frontend' => array(
    'default' => array(
      'backend' => 'Cm_Cache_Backend_Redis',
      'backend_options' => array(
        'server' => 'redis',
        'port' => '6379',
        'persistent' => '', // Specify a unique string like "cache-db0" to enable persistent connections.
        'database' => '0',
        'password' => '',
        'force_standalone' => '0', // 0 for phpredis, 1 for standalone PHP
        'connect_retries' => '1', // Reduces errors due to random connection failures
        'read_timeout' => '10', // Set read timeout duration
        'automatic_cleaning_factor' => '0', // Disabled by default
        'compress_data' => '1', // 0-9 for compression level, recommended: 0 or 1
        'compress_tags' => '1', // 0-9 for compression level, recommended: 0 or 1
        'compress_threshold' => '20480', // Strings below this size will not be compressed
        'compression_lib' => 'gzip', // Supports gzip, lzf and snappy,
        'use_lua' => '0' // Lua scripts should be used for some operations
      )
    ) ,
    'page_cache' => array(
      'backend' => 'Cm_Cache_Backend_Redis',
      'backend_options' => array(
        'server' => 'redis',
        'port' => '6379',
        'persistent' => '', // Specify a unique string like "cache-db0" to enable persistent connections.
        'database' => '1', // Separate database 1 to keep FPC separately
        'password' => '',
        'force_standalone' => '0', // 0 for phpredis, 1 for standalone PHP
        'connect_retries' => '1', // Reduces errors due to random connection failures
        'lifetimelimit' => '57600', // 16 hours of lifetime for cache record
        'compress_data' => '0' // DISABLE compression for EE FPC since it already uses compression
      )
    )
  )
) ,
```
and delete all Magento's cache with
```
rm -rf magento2/var/cache/*
```

## Enable Varnish
