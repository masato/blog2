title: "DockerのHTTP Routing - Part10: xip.io と Hipache で動的リバースプロキシ"
date: 2014-08-07 00:54:00
tags:
 - Docker
 - HTTPRouting
 - Hipache
 - xipio
 - Sinatra
 - Redis
description: confdとOpenRestyに続いてHipacheでも動的プロキシを試してみます。Hipacheは0.4.0を使います。GitHubからソースコードをcloneしてイメージをビルドしますが、このままビルドしたりhipacheをdocker pullしてもうまく動作しませんでした。もう少し親切なドキュメントかソースを修正してくれると助かるのですが。config.jsonのJSONでカンマが正しくなかったり、supervisord.confが私の環境だとuser=redisで起動しませんでした。
---

[confd](/2014/08/05/docker-reverse-proxy-xipio-nginx-confd-sinatra/)と[OpenResty](/2014/08/06/docker-reverse-proxy-xipio-openresty-sinatra/)に続いてHipacheでも動的プロキシを試してみます。

[Hipache](https://github.com/hipache/hipache)は0.4.0を使います。GitHubからソースコードをcloneしてイメージをビルドしますが、このままビルドしたり[hipache](https://registry.hub.docker.com/_/hipache/)を`docker pull`してもうまく動作しませんでした。
もう少し親切なドキュメントかソースを修正してくれると助かるのですが。
[config.json](https://github.com/hipache/hipache/blob/master/config/config.json)のJSONでカンマが正しくなかったり、[supervisord.conf](https://github.com/hipache/hipache/blob/master/supervisord.conf)が私の環境だと`user=redis`で起動しませんでした。

### TL;DR

DockerのHTTPルーターにconfdとOpenRestyとHipacheの3つを比べました。
Hipacheもよかったのですが、リバースプロキシした`RStudio Server`がなぜか認証エラーになったので、OpenRestyにしました。


<!-- more -->

### イメージのビルド

プロジェクトを作成して、`git clone`します。

``` bash
$ cd ~/docker_apps
$ git clone https://github.com/hipache/hipache.git
$ cd hipache
```

修正したDockerfileです。opensslをインストールしてオレオレ証明書も作成しました。

``` bash ~/docker_apps/Dockerfile
FROM ubuntu:14.04
MAINTAINER Masato Shimizu <ma6ato@gmail.com>

RUN sed -i~ -e 's/archive.ubuntu.com/ftp.jaist.ac.jp/' /etc/apt/sources.list \
 && apt-get update

RUN DEBIAN_FRONTEND=noninteractive apt-get -y install supervisor nodejs npm redis-server \
    openssl

RUN mkdir ./hipache
ADD . ./hipache

RUN npm install -g ./hipache --production

ENV NODE_ENV production

ADD ./supervisord.conf /etc/supervisor/conf.d/supervisord.conf

EXPOSE  80 443 6379

RUN cd /etc/ssl && openssl genrsa -out ssl.key 4096 \
 && openssl req -new -batch -key ssl.key -out ssl.csr \
 && openssl x509 -req -days 3650 -in ssl.csr -signkey ssl.key -out ssl.crt

CMD ["supervisord", "-n"]

```

supervisord.confも動作しなかったので、redisの起動ユーザーから`user=redis`を削除しました。
hipacheの設定ファイルもconfig.jsonを使うようにしています。

``` bash ~/docker_apps/supervisord.conf
[supervisord]
nodaemon=true

[program:hipache]
command=/usr/local/bin/hipache -c /usr/local/lib/node_modules/hipache/config/config.json
stdout_logfile=/var/log/supervisor/%(program_name)s.log
stderr_logfile=/var/log/supervisor/%(program_name)s.log
autorestart=true

[program:redis]
command=/usr/bin/redis-server
stdout_logfile=/var/log/supervisor/%(program_name)s.log
stderr_logfile=/var/log/supervisor/%(program_name)s.log
autorestart=true
```

config.jsonも余計なカンマを削除して、`accessLog`のパスも修正しています。
bindもローカルホストから、外部から接続可能にしました。

``` json  ~/docker_apps/config/config.json
{
    "server": {
        "accessLog": "/var/log/hipache.log",
        "workers": 10,
        "maxSockets": 100,
        "deadBackendTTL": 30,
        "tcpTimeout": 30,
        "retryOnError": 3,
        "deadBackendOn500": true,
        "httpKeepAlive": false
    },
    "https": {
        "port": 443,
        "bind": "0.0.0.0",
        "key": "/etc/ssl/ssl.key",
        "cert": "/etc/ssl/ssl.crt"
    },
    "http": {
        "port": 80,
        "bind": "0.0.0.0"
    },
    "driver": "redis://127.0.0.1:6379"
}
```

イメージをbuildします。

``` bash
$ docker build -t masato/hipache .
```

コンテナを起動します。

``` bash
$ docker run --name hipache -d -p 80:80 -p 443:443 masato/hipache
```

### redis-cliでルーティング設定

redis-cli用のコンテナを起動して、Redisの動作確認をします。

``` bash
$ docker run -it --rm --link hipache:redis dockerfile/redis bash -c 'redis-cli -h $REDIS_PORT_6379_TCP_ADDR'
172.17.1.36:6379> exit
```

`sinatra1.10.1.2.164.xip.io`と`sinatra2.10.1.2.164.xip.io`がそれぞれのSinatraコンテナにルーティングされるように設定します。

``` bash
$ docker run -it --rm --link hipache:redis dockerfile/redis bash -c 'redis-cli -h $REDIS_PORT_6379_TCP_ADDR rpush frontend:sinatra1.10.1.2.164.xip.io sinatra1'
(integer) 1
$ docker run -it --rm --link hipache:redis dockerfile/redis bash -c 'redis-cli -h $REDIS_PORT_6379_TCP_ADDR rpush frontend:sinatra1.10.1.2.164.xip.io http://172.17.0.37:5000'
(integer) 2
$ docker run -it --rm --link hipache:redis dockerfile/redis bash -c 'redis-cli -h $REDIS_PORT_6379_TCP_ADDR rpush frontend:sinatra2.10.1.2.164.xip.io sinatra2'
(integer) 1
$ docker run -it --rm --link hipache:redis dockerfile/redis bash -c 'redis-cli -h $REDIS_PORT_6379_TCP_ADDR rpush frontend:sinatra2.10.1.2.164.xip.io http://172.17.0.191:5000'
(integer) 2
```

Redisに登録されたルーティングを確認します。

``` bash
$ docker run -it --rm --link hipache:redis dockerfile/redis bash -c 'redis-cli -h $REDIS_PORT_6379_TCP_ADDR LRANGE 'frontend:sinatra1.10.1.2.164.xip.io' 0 -1'
1) "sinatra1"
2) "http://172.17.0.37:5000"
$ docker run -it --rm --link hipache:redis dockerfile/redis bash -c 'redis-cli -h $REDIS_PORT_6379_TCP_ADDR LRANGE 'frontend:sinatra2.10.1.2.164.xip.io' 0 -1'
1) "sinatra2"
2) "http://172.17.0.191:5000"
```

### 確認

curlでHTTPSの通信を確認します。オレオレ証明書なので`-k`オプションをつけます。

``` bash
$ curl -k https://sinatra1.10.1.2.164.xip.io 
Hello world!
$ curl -k https://sinatra2.10.1.2.164.xip.io
Hello world!
```

### まとめ

Hipacheのイメージを作るところまで時間がかかりましたが、オープンソースなので自分でソースを読みながら調べれば動かせます。

バックエンドのデータベースは、Redisの他に[etcd](https://github.com/jkingyens/hipache)も選べるようになりました。CoreOSにデプロイするときにつかってみようと思います。

`HTTP Routing`の実装を[OpenResty](/2014/08/06/docker-reverse-proxy-xipio-openresty-sinatra/)と[confd](/2014/08/05/docker-reverse-proxy-xipio-nginx-confd-sinatra/)とHipacheの3つを比べてみました。





