title: docker-confd-nginx-volume-nsenter
tags:
---

Supervisord

``` conf /etc/supervisor/conf.d/confd.conf
[program:confd]
command=/confd -node="http://127.0.0.1:4001" -verbose=true
priority=10
numprocs=1
autostart=true
autorestart=true
stdout_events_enabled=true
stderr_events_enabled=true
```
https://registry.hub.docker.com/u/aegypius/confd/
docker run -v /path/to/confd/config:/etc/confd -t aegypius/confd

https://registry.hub.docker.com/u/aegypius/nginx/

docker run -d -p 80:80 -v <sites-enabled-dir>:/etc/nginx/sites-enabled -v <log-dir>:/var/log/nginx aegypius/nginx

$ docker run --name confd-data -v /etc/confd -v /etc/nginx/sites-enabled -v /var/log/nginx busybox true

$ docker ps -a
CONTAINER ID        IMAGE                          COMMAND                CREATED             STATUS                      PORTS                    NAMES
47b53466f04e        busybox:latest                 true                   8 seconds ago       Exited (0) 8 seconds ago                             confd-data

``` bash
$ docker inspect 47b53466f04e | grep "vfs/dir"
        "/etc/confd": "/var/lib/docker/vfs/dir/d88d67bfd41b0335359338bb2a27d0c4e467f8ec78c5474bf7a2188eee2aa425",
        "/etc/nginx/sites-enabled": "/var/lib/docker/vfs/dir/2d2a5d3d5891a08f7922a9e219a545b633f5ef732967c247e9f80ef1ac9cec4d",
        "/var/log/nginx": "/var/lib/docker/vfs/dir/0768328f8183a1c84d9411baef8f41b9968c76bdbb9aa2ab059e8dafa96560c7"
```

$ docker run --name confd-nginx -d -p 80:80 --volumes-from confd-data aegypius/nginx
$ docker-nsenter d87bc1603
# vi /etc/nginx/site-enabled/default

server {
    listen 80 default_server;
    #listen [::]:80 default_server ipv6only=on;

    root /usr/share/nginx/html;
    index index.html index.htm;
    server_name localhost;

    location / {
        try_files $uri $uri/ /index.html;
    }
}



