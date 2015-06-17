title: 'Dockerで開発環境をつくる - Nginx Part2: Hello World, Again'
date: 2014-07-29 22:57:44
tags:
 - DockerDevEnv
 - Nginx
description: DockerのHTTP Routing - Part4 xip.io と Nginx と Node.jsのrunitで記述したdaemon off;はNginxをフォアグラウンドで実行してくれるので、DockerfileのCMDに書いて使えます。
---

* `Update 2014-08-04`: [Dockerで開発環境をつくる - Nginx Part3: Supervisorでデモナイズ](/2014/08/04/docker-devenv-nginx-supervisor/)

[DockerのHTTP Routing - Part4: xip.io と Nginx と Node.js](/2014/07/23/docker-reverse-proxy-xipio-nginx/)のrunitで記述した`daemon off;`はNginxをフォアグラウンドで実行してくれるので、DockerfileのCMDに書いて使えます。

### Dockerfile

``` bash ~/docker_apps/nginx-minimal/Dockerfile
FROM ubuntu:14.04
MAINTAINER Masato Shimizu <ma6ato@gmail.com>

RUN sed -i~ -e 's/archive.ubuntu.com/ftp.jaist.ac.jp/' /etc/apt/sources.list \
 && apt-get update

RUN DEBIAN_FRONTEND=noninteractive apt-get install -y nginx \
 && sed -i 's/^\t\(listen \[::\]:80 .*\)$/\t#\1/' /etc/nginx/sites-available/default \
 && echo 'Hello World, Again' > /usr/share/nginx/html/index.html
EXPOSE 80
CMD [ "nginx", "-g", "daemon off;" ]
```

### コンテナの起動と確認

イメージのビルドとコンテナを起動します。

``` bash
$ docker build -t masato/nginx-minimal .
$ docker run --rm --name nginx-minimal -it masato/nginx-minimal
```

新しいシェルを起動してcurlで確認します。

``` bash
$ curl $(docker inspect --format="&#123;&#123; .NetworkSettings.IPAddress }}" nginx-minimal)
Hello World, Again
```