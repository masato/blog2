title: "Dockerでデータ分析環境 - Part4: Tomcat7とPostgreSQL Studio"
date: 2014-07-13 11:38:16
tags:
 - Docker
 - AnalyticSandbox
 - PostgreSQL
 - Java
 - Tomcat
 - runit
description: PostgreSQLの管理ツールはPostgreSQL Studioを使います。データ分析のためSQLの学習環境を構築するのが目的なので、Webから便利にクエリが書けるを使います。PostgreSQL Studioの画面はいまだとちょっと懐かしいGWTで作られています。Scalaの開発環境は以前作ったのですが、まだDockerでTomcatの開発環境は作っていませんでした。
---

* `Update 2014-10-04`: [Docker開発環境をつくる - Emacs24とEclimからEclipseに接続するJava開発環境](/2014/10/04/docker-devenv-emacs24-eclim-java/)

PostgreSQLの管理ツールは[PostgreSQL Studio](http://www.postgresqlstudio.org/)を使います。
データ分析のためSQLの学習環境を構築するのが目的なので、Webから便利にクエリが書ける[SQL WorkSheet](http://www.postgresqlstudio.org/about/features/#worksheet)を使います。

`PostgreSQL Studio`の画面はいまだとちょっと懐かしいGWTで作られています。

[Scala](/2014/06/30/emacs-scala-typesafe-activator-ensime/)の開発環境は以前作ったのですが、まだDockerでTomcatの開発環境は作っていませんでした。

<!-- more -->

### Dockerfile

いつもの[phusion/baseimage](https://registry.hub.docker.com/u/phusion/baseimage/)を使います。
disposableなDockerだとwarファイルのデプロイは初回だけで済むので、繰り返しデプロイすることを考えなければ、
warファイルはwebappsにコピーするだけでデプロイできます。

昔はそれほど感じませんでしたが、warファイルのデプロイは便利です。


``` bash ~/docker_apps/postgresql-studio/Dockerfile
FROM phusion/baseimage:0.9.11

ENV HOME /root
RUN /etc/my_init.d/00_regen_ssh_host_keys.sh

## Install an SSH of your choice.
ADD google_compute_engine.pub /tmp/your_key
RUN cat /tmp/your_key >> /root/.ssh/authorized_keys \
  && chmod 600 /root/.ssh/authorized_keys \
  && chmod 600 /root/.ssh/authorized_keys && rm -f /tmp/your_key

# ...put your own build instructions here...
RUN sed -i~ -e 's/archive.ubuntu.com/ftp.jaist.ac.jp/' /etc/apt/sources.list \
 && apt-get update

## Japanese Environment
RUN DEBIAN_FRONTEND=noninteractive apt-get install -y language-pack-ja
ENV LANG ja_JP.UTF-8
RUN update-locale LANG=ja_JP.UTF-8
RUN mv /etc/localtime /etc/localtime.org
RUN ln -s /usr/share/zoneinfo/Asia/Tokyo /etc/localtime

## Development Environment
ENV EDITOR vim
RUN update-alternatives --set editor /usr/bin/vim.basic
RUN DEBIAN_FRONTEND=noninteractive \
    apt-get install -y software-properties-common git wget curl unzip

## Java, Tomcat7
RUN DEBIAN_FRONTEND=noninteractive add-apt-repository -y ppa:webupd8team/java \
  && apt-get -y update \
  && yes | apt-get install oracle-java8-installer
RUN DEBIAN_FRONTEND=noninteractive apt-get install -y tomcat7
ENV JAVA_HOME /usr/lib/jvm/java-8-oracle

## PostgreSQL Studio
RUN wget http://www.postgresqlstudio.org/?ddownload=47438 -O /root/pgstudio_1.2.zip \
  && unzip /root/pgstudio_1.2.zip -d /var/lib/tomcat7/webapps/

ADD sv /etc/service

# Use baseimage-docker's init system.
CMD ["/sbin/my_init"]

# Clean up APT when done.
RUN apt-get clean && rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*
```

### runitの設定ファイル

Tomcatの起動方法はすっかり忘れてしまっていました。実行時にいくつか環境変数を渡す必要があります。

``` bash ~/docker_apps/postgresql-studio/sv/tomcat7/run
#!/bin/sh

exec 2>&1
export CATALINA_HOME=/usr/share/tomcat7
export CATALINA_BASE=/var/lib/tomcat7
mkdir -p $CATALINA_BASE/temp

export JAVA_HOME=/usr/lib/jvm/java-8-oracle
exec /usr/share/tomcat7/bin/catalina.sh run
```

svlogdを使ったログの設定です。

``` bash ~/docker_apps/postgresqlsv/tomcat7/log/run
#!/bin/sh
service=$(basename $(dirname $(pwd)))
logdir="/var/log/${service}"
mkdir -p ${logdir}
exec 2>&1
exec /usr/bin/svlogd -tt ${logdir}
```

### イメージの build と run

イメージをビルドします。

``` bash
$ cd ~/docker_apps/postgresql-studio
$ docker build -t masato/postgresql-studio . 
```

コンテナを起動します。Tomcatの8080ポートをエクスポーズします。

``` bash
$ docker run --rm -i -t -p 8080:8080 masato/postgresql-studio /sbin/my_init bash
```

### ブラウザで確認

Dockerホストからブラウザを開いて、`PostgreSQL Studio`のURLを開きます。

[PostgreSQLコンテナ](/2014/07/12/docker-analytic-sandbox-postgresql/)への接続情報を入力します。

{% img center /2014/07/13/docker-analytic-sandbox-postgresql-studio/postgresql-studio-error.png %}

本当は5432以外を指定したいのですが、なぜか5432に自動的に補正されてしまいます。
PostgreSQLコンテナはまだ起動していないので、接続エラーになります。
残念ながらエラーメッセージも文字化けしてしまいました。

### Dockerコンテナにログインしてデバッグ

トップページのJSPを見ると、pageディレクティブにcontentTypeの指定がありません。

``` jsp /var/lib/tomcat7/webapps/pgstudio/PgStudio.jsp
<!doctype html>
<!-- The DOCTYPE declaration above will set the    -->
<!-- browser's rendering engine into               -->
<!-- "Standards Mode". Replacing this declaration  -->
<!-- with a "Quirks Mode" doctype may lead to some -->
<!-- differences in layout.                        -->
```

とりあえずダウンロードしてきたwarファイルをunpackして、sedで編集後にpackすることにしました。

``` bash ~/docker_apps/postgresql-studio/Dockerfile
RUN cd /root && wget http://www.postgresqlstudio.org/?ddownload=47438 -O ./pgstudio_1.2.zip \
  && unzip ./pgstudio_1.2.zip  && unzip pgstudio.war -d ./pgstudio \
  && sed -i '1s/^/<%@ page language="java" contentType="text\/html; charset=UTF-8" pageEncoding="UTF-8" %>\n/' ./pgstudio/PgStudio.jsp \
  && cd ./pgstudio && jar -cvf /var/lib/tomcat7/webapps/pgstudio.war .
```

イメージをビルドし直して、コンテナを起動します。文字化けは解消されました。

``` bash
$ docker build -t masato/postgresql-studio .
$ docker run --rm -i -t -p 8080:8080 masato/postgresql-studio  /sbin/my_init bash
```


{% img center /2014/07/13/docker-analytic-sandbox-postgresql-studio/postgresql-studio.png %}

### PostgreSQLのコンテナへ接続

PostgreSQLコンテナをバックグラウンドで起動します。

```
$ docker run -d --name="postgresql" \
             -p 5432:5432 \
             -v /opt/docker_vols/postgresql:/data \
             -e USER="masato" \
             -e DB="spike_db" \
             -e PASS="password" \
             paintedfox/postgresql
```

`PostgreSQL Studio`からもこの接続情報でログインできるようになりました。

{% img center /2014/07/13/docker-analytic-sandbox-postgresql-studio/postgresql-studio-login.png %}


### まとめ

データ分析環境として、基本のR,Python,SQLをWebから実行できるDockerイメージができました。

[Typsafe Activator](/2014/06/30/emacs-scala-typesafe-activator-ensime/)もWebからコードを実行できる環境なのですが、Scalaも書いてみたい人がいれば追加しようと思います。

この後は本番用のDockerホストと、フロントエンドにHipacheかHAProxyでリバースプロキシを用意して、`HTTP Routing`できるようにしようと思います。






