title: "Switching Docker Reverse Proxy from dockge-gen to OpenResty"
date: 2014-11-14 23:29:07
tags:
 - OpenResty
 - Nginx
 - docker-gen
 - HTTPRouting
description: I give up to continue to use a docker-gen, although it is convenient for us to create a Nginx config file automatically. If a docker-gen unit has been killed every other related units are killed. I decide to switch our reverse proxy from docker-gen and nginx to OpenResty and Redis combination.
---


 I give up to continue to use a docker-gen, although it is convenient for us to create a Nginx config file automatically. If a docker-gen unit has been killed every other related units are killed. I decide to switch our reverse proxy from docker-gen and nginx to OpenResty and Redis combination.

<!-- more -->

### Create Project

The openresty-moin project directory contains Dockerfile which extends [3scale/openresty](https://github.com/3scale/docker-openresty). The directory structure is below.

``` bash
$ tree .~/docker_apps/openresty-moin
~/docker_apps/openresty-moin
├── Dockerfile
├── nginx
│   ├── certs
│   │   ├── ssl.crt
│   │   └── ssl.key
│   └── nginx.conf
└── openresty.conf
```

### Dockerfile

The Dockerfile is simply extends parent image. Allowing access from remote redis-client, I use sed command to  `/etc/redis/redis.conf`.

``` bash Dockerfile
FROM 3scale/openresty
MAINTAINER Masato Shimizu <ma6ato@gmail.com>
RUN mkdir -p /opt/nginx/certs && mkdir -p /opt/nginx/www
RUN sed -i 's/bind 127.0.0.1/bind 0.0.0.0/g' /etc/redis/redis.conf
ADD openresty.conf /etc/supervisor/conf.d/openresty.conf
ADD nginx /opt/nginx
VOLUME ["/opt/nginx"]
EXPOSE 80 443 6379
```

Because this container contains SSL Certificate, we shoud put a Docker images to a private registry.

### openresty.conf

The openresty.conf file is copied to '/etc/supervisor/conf.d' and used as a Supervispr config file. This config file runs Nginx in foregroud.

``` bash openresty.conf
[program:openresty]
command=/opt/openresty/nginx/sbin/nginx -p /opt/nginx/www -c /opt/nginx/nginx.conf -g 'daemon off;'
autorestart=true
```

### nginx.conf

The nginx.conf file is used in the Supvervisor config file with -c flag. This nginx.conf forces HTTP connections redirect to HTTPS. Using a enbedded Lua script it determines a upstream URL from Reds record dynamically.

``` bash nginx.conf
worker_processes  1;
error_log  /dev/stderr debug;
events {
    worker_connections  256;
}
http {
    server {
      listen 80;
      return 301 https:/$host$request_uri;
    }
    server {
        listen  443 ssl deferred;
        client_max_body_size 20M;
        ssl on;
        ssl_certificate /opt/nginx/certs/ssl.crt;
        ssl_certificate_key /opt/nginx/certs/ssl.key;
        error_log /proc/self/fd/2;
        access_log /proc/self/fd/1;
        server_name  localhost;
        proxy_buffer_size 64K;
        proxy_buffers 32 48K;
        proxy_busy_buffers_size 256K;
        ssl_protocols        SSLv3 TLSv1 TLSv1.1 TLSv1.2;
        ssl_ciphers RC4:HIGH:!aNULL:!MD5;
        ssl_prefer_server_ciphers on;
        keepalive_timeout    60;
        ssl_session_cache    shared:SSL:10m;
        ssl_session_timeout  10m;
        proxy_next_upstream error timeout invalid_header http_500 http_502 http_503 http_504;
        proxy_set_header        Accept-Encoding   "";
        proxy_set_header        Host            $host;
        proxy_set_header        X-Real-IP       $remote_addr;
        proxy_set_header        X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header        X-Forwarded-Proto $scheme;
        add_header              Front-End-Https   on;
        proxy_redirect     off;
        location / {
            set $upstream "";
            rewrite_by_lua '
               local res = ngx.location.capture("/redis")
               if res.status == ngx.HTTP_OK then
                  ngx.var.upstream       = res.body
               else
                  ngx.exit(ngx.HTTP_FORBIDDEN)
               end
            ';
            proxy_pass http://$upstream;
        }
        location /redis {
             internal;
             set            $redis_key $host;
             redis_pass     127.0.0.1:6379;
             default_type   text/html;
        }
    }
}
```

### OpenResty unit file 

Finally I cerate a OpenResty unit file which replaces docker-gen and Nginx unit files.  In order to avoid  docker-gen related complex unit dpendencies, I compile a handy single OpenResty container.

``` bash  openresty-moin@.service
[Unit]
Description=OpenResty Service
Requires=moinmoin@%i.service
After=moinmoin@%i.service
[Service]
TimeoutStartSec=0
ExecStartPre=-/usr/bin/docker kill %p%i
ExecStartPre=-/usr/bin/docker rm -v %p%i
ExecStartPre=/usr/bin/docker pull xxx.xxx.xxx.xxx:5000/openresty-moin
ExecStart=/usr/bin/docker run --name %p%i \
  -p 443:443 \
  xxx.xxx.xxx.xxx:5000/openresty-moin
ExecStop=/usr/bin/docker kill %p%i
[Install]
WantedBy=multi-user.target
[X-Fleet]
Conflicts=%p@*.service
MachineMetadata=role=moin
```


