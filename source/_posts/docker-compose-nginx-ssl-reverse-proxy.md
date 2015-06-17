title: "Docker Composeを使ってNginxのSSLリバースプロキシを起動する"
date: 2015-04-23 17:24:30
categories:
 - Docker
tags:
 - Docker
 - DockerCompose
 - Nginx
 - OpenResty
 - Nodejs
 - node-static
description: Nginxのnginx.confではLuaなどを使わないと環境変数を読み込めません。Dockerのリンクを使う場合はコンテナの起動時に設定された環境変数をsedなどで置換する起動スクリプトを用意してnginx.confを使っていました。Docker Composeのlinksを使うとコンテナの/etc/hostsにエントリを作成してくれます。
---

Nginxの`nginx.conf`ではLuaなどを使わないと環境変数を読み込めません。Dockerのリンクを使う場合はコンテナの起動時に設定された環境変数をsedなどで置換する起動スクリプトを用意して`nginx.conf`を使っていました。Docker Composeの`links`を使うとコンテナの`/etc/hosts`に[エントリを作成](http://docs.docker.com/compose/yml/#links)してくれます。

<!-- more -->

## OpenResty

Nginxのリバースプロキシは後でLuaでHTTPヘッダの制御やAPIのアクセスコントロールをしたいのでOpenRestyを使います。Dockerイメージは[tenstartups/openresty](https://registry.hub.docker.com/u/tenstartups/openresty/)がミニマルで良さそうです。


## コンテナを個別に起動する場合

最初にDocker Composeではなく通常のDockerの使い方でリバースプロキシとHTTPサーバーを`docker run`でそれぞれ起動します。
OpenRestyで使うSSL証明書はDockerホスト上に自己署名で作成します。

``` bash
$ mkdir -p /opt/nginx/certs
$ cd /opt/nginx/certs
$ openssl genrsa -out server.key 4096
$ openssl req -new -batch -key server.key -out server.csr
$ openssl x509 -req -days 3650 -in server.csr -signkey server.key -out server.crt
```

`nginx.conf`も同様にDockerホスト上に用意してコンテナの`/opt/nginx`ディレクトリをコンテナにマウントします。`proxy_pass`のプロキシ先は[node-static](https://github.com/cloudhead/node-static)のコンテナを用意します。node-staciはDockerホストにポートマップします。今回は確認用なのでDockerホストを固定で指定しておきます。


``` bash /opt/nginx/nginx.conf
daemon off;
worker_processes  1;

events {
    worker_connections  256;
}

http {
    server {
        listen 80;
        return 301 https://$host$request_uri;
    }

    server {

        listen 443;
        server_name www.example.com;

        access_log /proc/self/fd/1;
        error_log /proc/self/fd/2;

        ssl_certificate           /etc/nginx/certs/server.crt;
        ssl_certificate_key    /etc/nginx/certs/server.key;

       ssl on;
       ssl_prefer_server_ciphers on;
       ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
       ssl_ciphers "ECDHE+RSAGCM:ECDH+AESGCM:DH+AESGCM:ECDH+AES256:DH+AES256:ECDH+AES128:DH+AES:!EXPORT:!DES:!3DES:!MD5:!DSS";

        location / {
            proxy_set_header        Host $host;
            proxy_set_header        X-Real-IP $remote_addr;
            proxy_set_header        X-Forwarded-Host $host;
            proxy_set_header        X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header        X-Forwarded-Proto $scheme;
            proxy_pass                 http://10.3.0.165:8080;
            proxy_redirect http:// https://;
        }
    }
}
```

### HTTPサーバー (node-static)

node-staticのDockerイメージを用意します。

``` bash ~/node_apps/node-static/Dockerfile
FROM google/nodejs-runtime
VOLUME /app/public
```

`app.js`は8080ポートでLISTENして起動します。

``` js ~/node_apps/node-static/app.js
var static = require('node-static');
var file = new static.Server('./public');

require('http').createServer(function (request, response) {
    request.addListener('end', function () {
        file.serve(request, response);
    }).resume();
}).listen(8080);

console.log("Server running at http://localhost:8080");
```

`package.json`からnode-staticのパッケージをインストールします。

``` json ~/node_apps/node-static/package.json
{
  "name": "node-static-app",
  "description": "node-static app",
  "version": "0.0.1",
  "private": true,
  "dependencies": {
    "node-static": "0.7.6"
  },
  "scripts": {"start": "node app.js"}
}
```

`index.html`はふつうのHello Worldです。

``` html ~/node_apps/node-static/public/index.html
<!DOCTYPE html>
<html lang="ja">
  <head>
    <meta charset="utf-8">
    <title>hello world</title>
  </head>
  <body>
   <p>hello world</p>
  </body>
</html>
```

node-staticのイメージをビルドして起動します。

``` bash
$ cd  ~/node_apps/node-static/p
$ docker pull google/nodejs-runtime
$ docker build -t node-static .
$ docker run --name node-static \
  -d \
  -p 8080:8080 \
  -v $PWD/public:/app/public \
  node-static
```

openrestyのコンテナを起動します。

``` bash
$ docker pull tenstartups/openresty
$ docker run --name nginx \
  -d \
  -p 80:80 \
  -p 443:443 \
  -v /opt/nginx:/etc/nginx \
  tenstartups/openresty
```

Dockerコンテナからcurlに`--insecure`フラグを付けてOpenRestyがSSLターミネーションとリバースプロキシが動作していることを確認します。

``` bash
$ curl https://localhost --insecure
<!DOCTYPE html>
<html lang="ja">
  <head>
    <meta charset="utf-8">
    <title>hello world</title>
  </head>
  <body>
   <p>hello world</p>
  </body>
</html>
```


## Docker Composeを使う場合

次にDocker Composeを使って2つのコンテナをオーケストレーションしてみます。`docker-compose.yml`を用意します。`openresty`コンテナに`nodestatic`コンテナをリンクします。

```yaml ~/node_apps/node-static/docker-compose.yml
nodestatic:
  restart: always
  build: .
openresty:
  restart: always
  image: tenstartups/openresty
  ports:
    - "80:80"
    - "443:443"
  volumes:
    - /opt/nginx:/etc/nginx 
  links:
    - node-static
```

Docker Composeのlinksを使うとコンテナの`/etc/hosts`経由で名前解決できるようになります。`nginx.conf`のプロキシ先のホスト名をDocker Composeでリンクしたコンテナの名前に変更します。

``` bash /opt/nginx/nginx.conf
            proxy_pass                 http://nodestatic:8080;
```

`docker-compose`コマンドから2つのコンテナをupします。

```bash
$ docker-compose up -d
Creating nodestatic_nodestatic_1...
Creating nodestatic_openresty_1...
```

以下のようにコンテナが起動しました。

``` bash
$ docker-compose ps
         Name                     Command                    State                     Ports
-----------------------------------------------------------------------------------------------------
nodestatic_nodestatic_1   /nodejs/bin/npm start     Up                        8080/tcp
nodestatic_openresty_1    ./entrypoint nginx -c     Up                        0.0.0.0:443->443/tcp,
                          /etc ...                                            0.0.0.0:80->80/tcp
```

Dockerホストからcurlでリバースプロキシを確認します。

``` bash
$ curl https://localhost --insecure
<!DOCTYPE html>
<html lang="ja">
  <head>
    <meta charset="utf-8">
    <title>hello world</title>
  </head>
  <body>
   <p>hello world</p>
  </body>
</html>
```
