title: 'Dockerで開発環境をつくる - Nginx Part3: Supervisorでデモナイズ'
date: 2014-08-04 19:42:14
tags:
 - DockerDevEnv
 - Nginx
 - DockerfileProject
 - Supervisor
description: Dockerで開発環境をつくる - Nginx Part2 Hello World, Againで作成したDockerfileはCMDでnginxを起動してるので複数のサービスを実行できません。そこでSupervisorでデモナイズするように修正します。Dockerで開発環境をつくる - Nginx Part1のbaseimageはphusion/baseimageを使い、runitでした。今回のbaseimageはDockerfile Projectのdockerfile/supervisorなので、Supervisorを使います。
---

[Dockerで開発環境をつくる - Nginx Part2: Hello World, Again](/2014/07/29/docker-devenv-nginx-hello-world-again/)で作成したDockerfileはCMDでnginxを起動してるので複数のサービスを実行できません。
そこでSupervisorでデモナイズするように修正します。

[Dockerで開発環境をつくる - Nginx Part1](/2014/07/28/docker-devenv-nginx/)のbaseimageは[phusion/baseimage](https://registry.hub.docker.com/u/phusion/baseimage/)を使い、runitでした。

今回のbaseimageは[Dockerfile Project](http://dockerfile.github.io/)の[dockerfile/supervisor](https://registry.hub.docker.com/u/dockerfile/supervisor/dockerfile)なので、Supervisorを使います。


<!-- more -->

### Dockerfile

プロジェクトを作成します。

``` bash
$ mkdir -p ~/docker_apps/nginx
$ cd !$
```

今回のbaseimageは[Dockerfile Project](http://dockerfile.github.io/)の[dockerfile/supervisor](https://registry.hub.docker.com/u/dockerfile/supervisor/dockerfile)なので、Supervisorを使います。

``` bash ~/docker_apps/nginx/Dockerfile
FROM dockerfile/supervisor
MAINTAINER Masato Shimizu <ma6ato@gmail.com>

# apt-get update
RUN sed -i~ -e 's/archive.ubuntu.com/ftp.jaist.ac.jp/' /etc/apt/sources.list \
 && apt-get update

# nginx install
RUN DEBIAN_FRONTEND=noninteractive apt-get install -y software-properties-common \
    && add-apt-repository -y ppa:nginx/stable \
    && apt-get update \
    && apt-get install -y nginx \
    && sed -i 's/^\t\(listen \[::\]:80 .*\)$/\t#\1/' /etc/nginx/sites-available/default
ADD nginx.conf /etc/supervisor/conf.d/nginx.conf
RUN echo 'Hello World, Supervisor' > /usr/share/nginx/html/index.html

VOLUME ["/data", "/etc/nginx/sites-enabled", "/var/log/nginx"]

CMD ["supervisord", "-c", "/etc/supervisor/supervisord.conf"]

EXPOSE 80
```

`/etc/supervisor/conf.d`に配置するSupervisorのNginx設定ファイルです。

``` conf ~/docker_apps/nginx/nginx.conf 
[program:nginx]
command=/usr/sbin/nginx -g "daemon off;"
numprocs=1
autostart=true
autorestart=true
```

### ビルドと確認

イメージをビルドします。

``` bash
$ docker build -t masato/nginx .
```

コンテナを実行します。

``` bash
$ docker run --rm  -p 80:80 masato/nginx
2014-08-04 11:53:32,005 CRIT Supervisor running as root (no user in config file)
2014-08-04 11:53:32,005 WARN Included extra file "/etc/supervisor/conf.d/nginx.conf" during parsing
2014-08-04 11:53:32,028 INFO RPC interface 'supervisor' initialized
2014-08-04 11:53:32,028 CRIT Server 'unix_http_server' running without any HTTP authentication checking
2014-08-04 11:53:32,028 INFO supervisord started with pid 1
2014-08-04 11:53:33,030 INFO spawned: 'nginx' with pid 16
2014-08-04 11:53:34,042 INFO success: nginx entered RUNNING state, process has stayed up for > than 1 seconds (startsecs)
```

ブラウザで確認します。

{% img center /2014/08/04/docker-devenv-nginx-supervisor/nginx-supervisor.png %}


### まとめ

[nsenter](/2014/08/03/docker-volume-nsenter-mount/)が使えるようになるとsshdを省略できるのでDockerfileもシンプルになります。

今度はsystemdでCoreOSのUnitファイルを作成して、CoreOSクラスタにデプロイしてみます。



