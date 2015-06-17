title: "DockerのHTTP Routing - Part9: xip.io と OpenResty で動的リバースプロキシ"
date: 2014-08-06 00:37:39
tags:
 - Docker
 - HTTPRouting
 - Nginx
 - OpenResty
 - xipio
 - Sinatra
 - Lua
 - Gin
 - Redis
 - API開発
description: DockerのHTTP Routing - Part8 xip.io と Nginx と confd 0.6 で動的リバースプロキシではconfdを使い、動的リバースプロキシのサンプルをつくりました。今回は同じことをOpenRestyで試してみます。DockerのHTTP Routing - Part3 OpenResty はじめにでは3scale/openrestyの使い方がわからなかったのですが、いろいろ勉強していくと少しずつ理解できるようになりました。OpenResty(Nginx+Lua+Redis)という組み合わせはおもしろく、GinでJSON-APIサーバーが書けるように勉強したいと思います。
---

* `Update 2014-08-07`: [DockerのHTTP Routing - Part10: xip.io と Hipache で動的リバースプロキシ](/2014/08/07/docker-reverse-proxy-xipio-hipache-sinatra/)

[DockerのHTTP Routing - Part8: xip.io と Nginx と confd 0.6 で動的リバースプロキシ](/2014/08/05/docker-reverse-proxy-xipio-nginx-confd-sinatra/)ではconfdを使い、動的リバースプロキシのサンプルをつくりました。

今回は同じことをOpenRestyで試してみます。[DockerのHTTP Routing - Part3: OpenResty はじめに](/2014/07/21/docker-reverse-proxy-openresty/)では[3scale/openresty](https://github.com/3scale/docker-openresty)の使い方がわからなかったのですが、いろいろ勉強していくと少しずつ理解できるようになりました。

OpenResty(Nginx+Lua+Redis)という組み合わせはおもしろく、[Gin](http://gin.io/)でJSON-APIサーバーが書けるように勉強したいと思います。

<!-- more -->

### 3scale/openresty

[3scale/openresty]()をベースイメージにしてDockerfileを作成します。

まずはプロジェクトを作成します。

``` bash
$ mkdir ~/docker_apps/openresty
$ cd !$
```

Dockerfileです。

``` bash ~/docker_apps/openresty/Dockerfile
FROM 3scale/openresty

MAINTAINER Masato Shimizu <ma6ato@gmail.com>

RUN useradd --system nginx
ADD nginx.conf /var/www/
ADD openresty.conf /etc/supervisor/conf.d/openresty.conf

RUN sed -i 's/bind 127.0.0.1/bind 0.0.0.0/g' /etc/redis/redis.conf
EXPOSE 6379 80
```

Supervisorの設定ファイルです。

``` bash ~/docker_apps/openresty/openresty.conf
[program:openresty]
command=/opt/openresty/nginx/sbin/nginx -p /var/www/ -c nginx.conf
autorestart=true
```

Nginxの設定ファイルです。
`rewrite_by_lua`の箇所でRedisからルーティング情報を取得して、upstreamの値にセットしています。
こちらの[Gist](https://gist.github.com/hiboma/1670088)を参考にしました。
`proxy_set_header`を設定しないと、リダイレクトしてしまうので追加します。

Luaで簡単にNginxの機能拡張ができます。個人的には発想の転換になりました。

``` bash ~/docker_apps/openresty/nginx.conf
worker_processes  1;
error_log  /dev/stderr debug;
daemon off;
events {
    worker_connections  256;
}

http {
    server {
        listen       80 deferred;
        server_name  localhost;

        proxy_buffer_size 64K;
        proxy_buffers 32 48K;
        proxy_busy_buffers_size 256K;
        proxy_set_header Host $http_host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

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

### イメージのビルドとコンテナの実行

イメージをビルドします。

``` bash
$ docker build -t masato/openresty .
```

OpenRestyコンテナを起動します。

``` bash
$ docker run --name openresty -i -t -d -p 80:80 masato/openresty
```

### ルーティング情報の設定

`--rm`と`--link`オプションで、Dockerホストからワンショットのredis-cli用のコンテナを使ってルーティングをセットします。disposableでとてもDockerらしい使い方で気に入っています。

``` bash
$ docker run -it --rm --link openresty:redis dockerfile/redis bash -c 'redis-cli -h $REDIS_PORT_6379_TCP_ADDR set p5001.10.1.2.164.xip.io 172.17.0.37:5000'
OK
$ docker run -it --rm --link openresty:redis dockerfile/redis bash -c 'redis-cli -h $REDIS_PORT_6379_TCP_ADDR set p5002.10.1.2.164.xip.io 172.17.0.191:5000'
```

### 確認

Dockerホストからcurlで確認します。

``` bash
$ curl p5001.10.1.2.164.xip.io
Hello world!
$ curl p5002.10.1.2.164.xip.io
Hello world!
```

### まとめ

confdよりも慣れているRedisを使うこともあり、OpenRestyの方が好みです。
HipacheもバックエンドにRedisを使うので同じ感じに使えそうです。

confdの場合は設定ファイルが書き換わるかちょっと不安になります。
この前のバージョンではテンプレートに表現力が足りないので、サブドメイン毎に設定ファイルを書くことになり、
あまり動的な感じがしません。

逆に静的ファイルを作成してしまった後は、ルーティング情報はデータベースを持たないので安心だったりもします。

最後にHipacheの動的リバースプロキシを試してみて、どれを採用しようか決めようと思います。

