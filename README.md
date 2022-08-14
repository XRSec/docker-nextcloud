# Docker 部署 Nextcloud LADP 四件套

Blog：[Docker 部署 Nextcloud LADP 四件套](https://xrsec.vercel.app/Docker%20%E9%83%A8%E7%BD%B2%20Nextcloud%20LADP%20%E5%9B%9B%E4%BB%B6%E5%A5%97.html#Dockerfile)

## Dockerfile

```dockerfile
version: '2'

services:
    db:
        hostname: mfs_db
        image: mariadb
        container_name: mfs_db
        restart: on-failure:3
        volumes:
            - ./mysql/db:/var/lib/mysql
        command: --transaction-isolation=READ-COMMITTED --binlog-format=ROW
        environment:
            - MYSQL_ROOT_PASSWORD=
            - MYSQL_PASSWORD=
            - MYSQL_DATABASE=nextcloud
            - MYSQL_USER=nextcloud

    app:
        hostname: mfs_app
        container_name: mfs_app
        image: nextcloud:fpm
        links:
            - db
        restart: on-failure:3
        volumes:
            - ./nginx/nextcloud:/var/www/html
            - /etc/ssl/nas.crt:/etc/ssl/nas.crt
            - /etc/ssl/nas.key:/etc/ssl/nas.key
            - ./uploads.ini:/usr/local/etc/php/conf.d/uploads.ini
            - ./sources.list:/etc/apt/sources.list
            - ./redis/5.3.2.zip:/root/5.3.2.zip
            - ./nginx/345:/var/www/345
        environment:
            - MYSQL_ROOT_PASSWORD=
            - MYSQL_PASSWORD=
            - MYSQL_DATABASE=nextcloud
            - MYSQL_USER=nextcloud_PASSWORD=j7ba7f2fMcBanJou

    web:
        hostname: mfs_nginx
        container_name: mfs_nginx
        image: nginx
        restart: on-failure:3
        ports:
            - 33014:443
            - 33014:443/udp
        links:
            - app
        volumes:
            - ./nginx/conf/default.conf:/etc/nginx/conf.d/default.conf:rw
            - ./sources.list:/etc/apt/sources.list
        volumes_from:
            - app

    redis:
        hostname: mfs_redis
        container_name: mfs_redis
        image: redis
        restart: on-failure:3
        volumes:
            - ./redis/redis.conf:/etc/redis.conf:rw
            - ./redis/data:/data
        command: /usr/local/etc/redis/redis.conf

# apt install net-tools # ifconfig
# apt install iputils-ping # ping
# ----------------------------------------
# docker exec -it mfs_app /bin/bash
# docker-php-source extract
# chmod 777 /var/www/html && chown www-data:www-data /var/www/html
# apt update && apt install imagemagick unzip -y
# docker-php-ext-install mysqli
# cd /root && unzip 5.3.2.zip
# mv phpredis-5.3.2/ /usr/src/php/ext/phpredis
# 记得去redis/redis.conf修改密码 requirepass
```

命令：`docker-compose up -d `



## Nginx

这个可以借鉴一下官方的

```conf
upstream php-handler {
    server mfs_app:9000;
    # server unix:/var/run/php/php7.4-fpm.sock;
}

# server {
#     listen 80;
#     listen [::]:80;
#     server_name cloud.hakase-labs.io;
#     # enforce https
#     return 301 https://$server_name:443$request_uri;
# }

server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name mfs_app.io;

    # Use Mozilla's guidelines for SSL/TLS settings
    # https://mozilla.github.io/server-side-tls/ssl-config-generator/
    # NOTE: some settings below might be redundant
    ssl_certificate /etc/ssl/nas.crt;
    ssl_certificate_key /etc/ssl/nas.key;

    # Add headers to serve security related headers
    # Before enabling Strict-Transport-Security headers please read into this
    # topic first.
    #add_header Strict-Transport-Security "max-age=15768000; includeSubDomains; preload;" always;
    #
    # WARNING: Only add the preload option once you read about
    # the consequences in https://hstspreload.org/. This option
    # will add the domain to a hardcoded list that is shipped
    # in all major browsers and getting removed from this list
    # could take several months.
    add_header Referrer-Policy "no-referrer" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-Download-Options "noopen" always;
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Permitted-Cross-Domain-Policies "none" always;
    add_header X-Robots-Tag "none" always;
    add_header X-XSS-Protection "1; mode=block" always;

    # Remove X-Powered-By, which is an information leak
    fastcgi_hide_header X-Powered-By;

    # Path to the root of your installation
    root /var/www/html;

    location = /robots.txt {
        allow all;
        log_not_found off;
        access_log off;
    }

    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   /var/www/345;
    }
    error_page 403 /403.html;
        location = /403.html {
          root  /var/www/345;
    }
    error_page 404 /404.html;
        location = /404.html {
          root  /var/www/345;
    }

    # The following 2 rules are only needed for the user_webfinger app.
    # Uncomment it if you're planning to use this app.
    #rewrite ^/.well-known/host-meta /public.php?service=host-meta last;
    #rewrite ^/.well-known/host-meta.json /public.php?service=host-meta-json last;

    # The following rule is only needed for the Social app.
    # Uncomment it if you're planning to use this app.
    #rewrite ^/.well-known/webfinger /public.php?service=webfinger last;

    location = /.well-known/carddav {
      return 301 $scheme://$host:$server_port/remote.php/dav;
    }
    location = /.well-known/caldav {
      return 301 $scheme://$host:$server_port/remote.php/dav;
    }

    # set max upload size
    client_max_body_size 512M;
    fastcgi_buffers 64 4K;

    # Enable gzip but do not remove ETag headers
    gzip on;
    gzip_vary on;
    gzip_comp_level 4;
    gzip_min_length 256;
    gzip_proxied expired no-cache no-store private no_last_modified no_etag auth;
    gzip_types application/atom+xml application/javascript application/json application/ld+json application/manifest+json application/rss+xml application/vnd.geo+json application/vnd.ms-fontobject application/x-font-ttf application/x-web-app-manifest+json application/xhtml+xml application/xml font/opentype image/bmp image/svg+xml image/x-icon text/cache-manifest text/css text/plain text/vcard text/vnd.rim.location.xloc text/vtt text/x-component text/x-cross-domain-policy;

    # Uncomment if your server is build with the ngx_pagespeed module
    # This module is currently not supported.
    #pagespeed off;

    location / {
        rewrite ^ /index.php;
    }

    location ~ ^\/(?:build|tests|config|lib|3rdparty|templates|data)\/ {
        deny all;
    }
    location ~ ^\/(?:\.|autotest|occ|issue|indie|db_|console) {
        deny all;
    }

    location ~ ^\/(?:index|remote|public|cron|core\/ajax\/update|status|ocs\/v[12]|updater\/.+|oc[ms]-provider\/.+)\.php(?:$|\/) {
        fastcgi_split_path_info ^(.+?\.php)(\/.*|)$;
        set $path_info $fastcgi_path_info;
        try_files $fastcgi_script_name =404;
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_param PATH_INFO $path_info;
        fastcgi_param HTTPS on;
        # Avoid sending the security headers twice
        fastcgi_param modHeadersAvailable true;
        # Enable pretty urls
        fastcgi_param front_controller_active true;
        fastcgi_pass php-handler;
        fastcgi_intercept_errors on;
        fastcgi_request_buffering off;
    }

    location ~ ^\/(?:updater|oc[ms]-provider)(?:$|\/) {
        try_files $uri/ =404;
        index index.php;
    }

    # Adding the cache control header for js, css and map files
    # Make sure it is BELOW the PHP block
    location ~ \.(?:css|js|woff2?|svg|gif|map)$ {
        try_files $uri /index.php$request_uri;
        add_header Cache-Control "public, max-age=15778463";
        # Add headers to serve security related headers (It is intended to
        # have those duplicated to the ones above)
        # Before enabling Strict-Transport-Security headers please read into
        # this topic first.
        #add_header Strict-Transport-Security "max-age=15768000; includeSubDomains; preload;" always;
        #
        # WARNING: Only add the preload option once you read about
        # the consequences in https://hstspreload.org/. This option
        # will add the domain to a hardcoded list that is shipped
        # in all major browsers and getting removed from this list
        # could take several months.
        add_header Referrer-Policy "no-referrer" always;
        add_header X-Content-Type-Options "nosniff" always;
        add_header X-Download-Options "noopen" always;
        add_header X-Frame-Options "SAMEORIGIN" always;
        add_header X-Permitted-Cross-Domain-Policies "none" always;
        add_header X-Robots-Tag "none" always;
        add_header X-XSS-Protection "1; mode=block" always;

        # Optional: Don't log access to assets
        access_log off;
    }

    location ~ \.(?:png|html|ttf|ico|jpg|jpeg|bcmap)$ {
        try_files $uri /index.php$request_uri;
        # Optional: Don't log access to other assets
        access_log off;
    }
}
```



## Redis

```conf
requirepass 你的密码
```

几千行，算了算了，全是注释



## Sources.list

```list
deb http://mirrors.aliyun.com/debian/ stretch main non-free contrib
deb-src http://mirrors.aliyun.com/debian/ stretch main non-free contrib
deb http://mirrors.aliyun.com/debian-security stretch/updates main
deb-src http://mirrors.aliyun.com/debian-security stretch/updates main
deb http://mirrors.aliyun.com/debian/ stretch-updates main non-free contrib
deb-src http://mirrors.aliyun.com/debian/ stretch-updates main non-free contrib
deb http://mirrors.aliyun.com/debian/ stretch-backports main non-free contrib
deb-src http://mirrors.aliyun.com/debian/ stretch-backports main non-free contrib
```



## Uploads.ini

```ini
file_uploads = On
memory_limit = 64M
upload_max_filesize = 64M
post_max_size = 64M
max_execution_time = 600
```



## 简单介绍

·# # 大家好

😅

为啥写了这一期呢 》因为之前手动装环境，被折磨的死去活来

写点建议给老司机

```shell
--privileged
/usr/sbin/init
```

- portainer

对了，怎么能少图呢？ 上才艺

## 概览

![image-20210114072550055](https://rmt.dogedoge.com/fetch/ZYGG/storage/20210114072554936230.png?w=1280&fmt=jpg)

![image-20210114072913027](https://rmt.dogedoge.com/fetch/ZYGG/storage/20210114072915201531.png?w=1280&fmt=jpg)

![image-20210114072944854](https://rmt.dogedoge.com/fetch/ZYGG/storage/20210114072946691796.png?w=1280&fmt=jpg)

![image-20210114073055620](https://rmt.dogedoge.com/fetch/ZYGG/storage/20210114073057809681.png?w=1280&fmt=jpg)

![image-20210114081611723](https://rmt.dogedoge.com/fetch/ZYGG/storage/20210114081614492156.png?w=1280&fmt=jpg)



![1](https://rmt.dogedoge.com/fetch/ZYGG/storage/20210115013459746771.jpeg?w=1280&fmt=jpg)



看下一期 自动备份 docker容器内容！