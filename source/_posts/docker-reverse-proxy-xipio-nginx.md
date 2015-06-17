title: "DockerのHTTP Routing - Part4: xip.io と Nginx と Node.js"
date: 2014-07-23 00:09:52
tags:
 - HTTPRouting
 - Nginx
 - nginx-proxy
 - Nodejs
 - xipio
 - runit
description: Passenger Nginx と Node.jsではマルチポートでLISTENするアプリをうまくリバースプロキシできませんでした。簡単なNode.jsアプリをNginxを動かすだけでも時間がかかってしまうので、もっと勉強しないといけないです。今回の目的は、xio.ioを使い、IPアドレスだけでWildcard DNSのテストをすることです。Namecheapで安いドメイン名を買えますが、開発や社内利用の場合、ローカルにDNSサーバーを用意しなくても名前解決ができるのはとても便利です。HTTP RoutingしたいDockerコンテナの各ポートを、Node.jsのアプリで見立て、動かしてみます。
---


* `Update 2014-07-28`: [Dockerで開発環境をつくる - Nginx](/2014/07/28/docker-devenv-nginx/)
* `Update 2014-07-29`: [Dockerで開発環境をつくる - Nginx Hello World, Again](/2014/07/29/docker-devenv-nginx-hello-world-again/)
* `Update 2014-08-05`: [DockerのHTTP Routing - Part8: xip.io と Nginx と confd 0.6 で動的リバースプロキシ](/2014/08/05/docker-reverse-proxy-xipio-nginx-confd-sinatra/)

[Passenger Nginx と Node.js](/2014/07/22/docker-devenv-passenger-nodejs/)ではマルチポートでLISTENするアプリをうまくリバースプロキシできませんでした。簡単なNode.jsアプリをNginxを動かすだけでも時間がかかってしまうので、もっと勉強しないといけないです。

今回の目的は、[xio.io](http://xip.io/)を使い、IPアドレスだけで`Wildcard DNS`のテストをすることです。Namecheapで安いドメイン名を買えますが、開発や社内利用の場合、ローカルにDNSサーバーを用意しなくても名前解決ができるのはとても便利です。
`HTTP Routing`したいDockerコンテナの各ポートを、Node.jsのアプリで見立て、動かしてみます。

<!-- more -->

### Dockerfile

最初に完成したDockerfileです。`phusion/baseimage`を使います。

``` bash ~/docker_apps/xipio/Dockerfile
FROM phusion/baseimage:0.9.11

# Set correct environment variables.
ENV HOME /root

# Use baseimage-docker's init process.
CMD ["/sbin/my_init"]

# ...put your own build instructions here...

# Regenerate SSH host keys.
RUN /etc/my_init.d/00_regen_ssh_host_keys.sh

# Install an SSH of your choice.
ADD my_key.pub /tmp/your_key
RUN cat /tmp/your_key >> /root/.ssh/authorized_keys && rm -f /tmp/your_key

# apt-get update
RUN sed -i~ -e 's/archive.ubuntu.com/ftp.jaist.ac.jp/' /etc/apt/sources.list \
 && apt-get update

# ppa
RUN apt-get install -y software-properties-common
RUN add-apt-repository ppa:chris-lea/node.js
RUN apt-get -yqq update

# apt-get install
RUN apt-get install -y nodejs nginx

# deploy Node.js app
RUN rm -rf /etc/nginx/sites-enabled/default
ADD webapp.conf /etc/nginx/sites-enabled/
RUN mkdir -p /srv/www/webapp/
ADD app.js /srv/www/webapp/
ADD sv /etc/service

# Clean up APT when done.
RUN apt-get clean && rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*
```

### runitの設定ファイル

Node.jsアプリのrunitの設定ファイルです。

``` bash ~/docker_apps/xipio/sv/nodejs/run
#!/bin/sh

cd /srv/www/webapp
exec 2>&1
exec chpst -U www-data node app.js > /var/log/nodejs.log 2>&1
```

runitのNginxです。

``` bash ~/docker_apps/xipio/sv/nginx/run
#!/bin/bash

exec 2>&1

exec /usr/sbin/nginx -g "daemon off;"
```

svlogdを使ったログの設定です。

``` bash ~/docker_apps/xipio/sv/nginx/log/run
#!/bin/sh

service=$(basename $(dirname $(pwd)))
logdir="/var/log/${service}"
mkdir -p ${logdir}
exec 2>&1
exec /usr/bin/svlogd -tt ${logdir}
```

### Node.jsアプリ

Node.jsで同一プロセスで複数ポートをLISTENするサンプルが[Can HttpServer listen to multiple ports?](https://groups.google.com/forum/#!topic/nodejs/EK53DqqUKHQ)にあったのでサンプルに使ってみます。

``` node ~/docker_apps/xipio/app.js
var http = require('http')
  , servers = [] 
  , port = 8000;

function addServer(fn) { 
    var s = http.createServer(fn);
    s.listen(port);
    servers.push(s); 
    port += 1;
} 

addServer(function(req,resp){ resp.writeHead(200); resp.end('hi'); }); 

addServer(function(req,resp){ resp.writeHead(200); resp.end('hello'); }); 

addServer(function(req,resp){ resp.writeHead(200); resp.end('howdy'); });
```

### xip.io用のNginxの設定

xip.ipのURLにポート番号を埋め込み、`server_name`の正規表現で動的にパースしてポート番号を抽出します。
リクエストに応じて動的にプロキシー先のポートを指定する仕組みになります。

``` conf ~/docker_apps/xipio/webapp.conf
server {
  listen 80;
  server_name ~^p(?<port_name>\d+)\.(.*)\.xip\.io$;

  location / {
    proxy_set_header host $host;
    proxy_set_header x-real-ip $remote_addr;
    proxy_set_header x-forwarded-for $proxy_add_x_forwarded_for;
    proxy_pass http://127.0.0.1:$port_name;
  }
}
```

### build

Dockerイメージをビルドします。

``` bash
$ cd ~/docker_apps/xipio
$ docker build -t masato/xipio .
```

### Node.jsを直接コールする

Node.jsのポートマップを3つ指定して、Dockerコンテナを実行します。

``` bash
$ docker run --rm -i -p 8000:8000 -p 8001:8001 -p 8002:8002 -t masato/xipio /sbin/
my_init bash
```

DockerホストのブラウザからNode.jsのポートを指定して確認します。

{% img center /2014/07/23/docker-reverse-proxy-xipio-nginx/nodejs.png %}


### xip.ioとNginxのリバースプロキシをつかう

DockerコンテナのIPアドレスを確認します。

``` bash
$ docker inspect b03feeae6063 | jq -r '.[0] | .NetworkSettings | .IPAddress'
172.17.0.207
```

Nginxの80だけをポートマップしてDockerコンテナを起動する。

``` bash
$ docker run --rm -i -p 80:80 -t masato/xipio /sbin/
my_init bash
```

Dockerホストから、xip.ioを使ったURLを指定してブラウザで確認します。

{% img center /2014/07/23/docker-reverse-proxy-xipio-nginx/xipio-nginx.png %}


### まとめ

xip.ioを使ったURLで、`Wildcard DNS`と`HTTP Routing`の確認ができたので社内用の[データ分析環境](/tags/AnalyticSandbox/)はxip.ioで束ねて使ってみます。
特にクラスタ環境を構築するときはローカルで名前解決が必要な場合が多いので、xip.ioはとても便利な仕組みです。


