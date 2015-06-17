title: "Dockerでデータ分析環境 - Part1: Ubuntu14.04にRStudio Serverをインストール"
date: 2014-07-09 22:20:29
tags:
 - Docker
 - AnalyticSandbox
 - Ubuntu
 - RStudioServer
 - R
description: 社内でデータ分析を勉強したいという声があがりました。声があがったのも驚きですが、ほとんどが非エンジニアのふつうの人で、それもRから勉強したいそうです。データ分析が一般業務として認知されはじめたようで、とてもうれしくなります。ふつうの人SSHやEmacsを使わず簡単にブラウザで勉強できる環境を用意しようと思います。こういった分析サンドボックスにこそ、Dockerがふさわしいので、さっそく`Rstudio Server`をUbuntu14.04にインストールしてみます。以前からApparmorとUpstartに悩まされ続けていて、Docker上で動かせませんでした。DruidのDockerfileんでいたらApparmorは無効にして、デモナイズはrunitを使っていたので、なるほどと思いました。Druidはちょっと触ってみたいのですが、クエリをJSONで書くみたいです。

---

* `Update 2014-08-08`: [Dockerでデータ分析環境 - Part5: RStudio ServerとData-onlyコンテナ](/2014/08/08/docker-analytic-sandbox-rstudio-data-only-container/)
* `Update 2014-08-09`: [Dockerでデータ分析環境 - Part6: Ubuntu14.04のDockerにRStudio ServerとOpenRestyをデプロイ](/2014/08/09/docker-analytic-sandbox-rstudio-openresty-deploy/)


社内でデータ分析を勉強したいという声があがりました。
声があがったのも驚きですが、ほとんどが非エンジニアのふつうの人で、それもRから勉強したいそうです。

データ分析が一般業務として認知されはじめたようで、とてもうれしくなります。
ふつうの人なので、SSHやEmacsを使わず簡単にブラウザで勉強できる環境を用意しようと思います。

こういった分析サンドボックスにこそ、Dockerがふさわしいので、さっそく`Rstudio Server`をUbuntu14.04にインストールしてみます。

以前からApparmorとUpstartに悩まされ続けていて、Docker上で動かせませんでした。
[Druid](https://github.com/metamx/druid)の[Dockerfile](https://github.com/mingfang/docker-druid/blob/master/Dockerfile)を読んでいたらApparmorは無効にして、デモナイズはrunitを使っていたので、なるほどと思いました。

Druidはちょっと触ってみたいのですが、クエリをJSONで書くみたいです。

<!-- more -->

### Dockerfile

いつものように、[phusion/baseimage](https://registry.hub.docker.com/u/phusion/baseimage/)を使います。versionは0.9.11です。Ubuntuは14.04になります。

``` bash ~/docker_apps/rstudio-server/Dockerfile

FROM phusion/baseimage:0.9.11

ENV HOME /root
RUN /etc/my_init.d/00_regen_ssh_host_keys.sh

## Install an SSH of your choice.
ADD mykey.pub /tmp/your_key
RUN cat /tmp/your_key >> /root/.ssh/authorized_keys \
  && chmod 600 /root/.ssh/authorized_keys \
  && chmod 600 /root/.ssh/authorized_keys && rm -f /tmp/your_key

## apt-get update
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
    apt-get install -y git wget curl unzip

## Install R
RUN echo 'deb http://cran.rstudio.com/bin/linux/ubuntu trusty/' > /etc/apt/sources.list.d/r.list && \
    apt-key adv --keyserver keyserver.ubuntu.com --recv-keys E084DAB9 && \
    apt-get update
RUN DEBIAN_FRONTEND=noninteractive apt-get install -y r-base fonts-takao

## RStudio Server
RUN DEBIAN_FRONTEND=noninteractive apt-get install -y gdebi-core libapparmor1 libcurl4-openssl-dev
RUN wget http://download2.rstudio.org/rstudio-server-0.98.953-amd64.deb && \
    gdebi -n rstudio-server-0.98.953-amd64.deb && \
    rm rstudio-server-*-amd64.deb
RUN useradd -m rstudio && \
    echo "rstudio:rstudio" | chpasswd

ADD sv /etc/service

## Use baseimage-docker's init system.
CMD ["/sbin/my_init"]

## Clean up APT when done.
RUN apt-get clean && rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*
```

### runitの設定ファイル

runitの設定ファイルは、[docker-druid](https://github.com/mingfang/docker-druid/blob/master/sv/rstudio-server/run)のrunを使わせてもらいます。

phusion/baseimageはrunitでデモナイズしているので、ちょうどよかったです。

``` bash ~/docker_apps/rstudio-server/sv/rstudio-server/run
#!/bin/sh

#sv start socklog-unix || exit 1

exec 2>&1
exec /usr/lib/rstudio-server/bin/rserver --server-daemonize=0 --server-app-armor-enabled=0
```

svlogdを使ったログの設定です。

``` bash ~/docker_apps/rstudio-server/sv/rstudio-server/log/run
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
$ cd ~/docker_apps/rstudio-server
$ docker build -t masato/rstudio-server . 
```

コンテナを起動します。SSHと`RStudio Server`のポートをエクスポーズします。

``` bash
$ docker run --rm -i -t  -p 22 -p 8788:8787 masato/rstudio-server  /sbin/my_init bash
```

### ブラウザで確認

別のワークステーションからブラウザを開いて、`RStudio Server`を使ってみます。

このバージョンは日本語の入出力と、変数名の日本語で定義できます。

* R: 3.1.0
* `RStudio Server`: 0.98.953

初心者向けのR本だと変数名に日本語を使っていることがあります。
Unicodeだと変数名に日本語が使えるのは理屈ではわかるのですが、とても抵抗がありました。
ふつうの人と話をしていると、変数？なので、「箱」です。というのはわかりやすのかも知れません。

{% img center /2014/07/09/docker-analytic-sandbox-rstudio-server/rstudio-server.png %}

### まとめ

とりあえず開発環境で`RStudio Server`のイメージができたので、あとでdocker-registryにpushして、
本番環境でpullしようと思います。

Rの他には、PythonとPostgreSQLの学習環境を用意して使ってもらおうと思います。
