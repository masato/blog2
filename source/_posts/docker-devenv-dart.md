title: 'Dockerで開発環境をつくる - Dartのインストール'
date: 2014-05-21 10:10:22
tags:
 - Dockerfile
 - DockerDevEnv
 - Dart
 - Ubuntu
description: この前はSSHで接続できるイメージを使ってDart用のコンテナを起動しましたが、普通にttyを開いてシェルを立ち上げた方が便利なので、Dockerfileをつかって起動します。調べているとDartの他にもElixirやScalaなどの開発環境を作っている人がいるので、いくつか試してみたいと思います。
---

[この前](/2014/05/18/docker-dart-redstone/)はSSHで接続できるイメージを使ってDart用のコンテナを起動しましたが、
普通にttyを開いてシェルを立ち上げた方が便利なので、Dockerfileをつかって起動します。
調べているとDartの他にも[Elixir](http://www.elixirdose.com/docker-as-elixir-development-environment/)や[Scala](http://tersesystems.com/2013/11/20/building-a-development-environment-with-docker/)などの開発環境を作っている人がいるので、いくつか試してみたいと思います。

<!-- more -->

### Dockerfileの作成
プロジェクトを作成します。
``` bash
$ mkdir -p ~/docker_apps/dart
$ cd !$
```
[Docker on Dart](https://github.com/sethladd/docker-dart)にあるDockerfileを使います。

``` ~/docker_apps/dart/Dockerfile
FROM ubuntu

RUN apt-get update
RUN apt-get install -y wget unzip
WORKDIR /tmp
RUN wget http://storage.googleapis.com/dart-archive/channels/stable/release/latest/sdk/dartsdk-linux-x64-release.zip
RUN unzip dartsdk-linux-x64-release.zip
RUN mv dart-sdk /opt
ENV PATH /opt/dart-sdk/bin:$PATH
```

Dockerfileからbuildして、イメージを作成します。
```
$ docker build -t masato/dart-sdk2 .
```

作成したイメージからコンテナを起動して、シェルを開きます。
``` bash
$ docker run -i -t masato/dart-sdk2 /bin/bash
# root@1c65b6c192eb:/tmp#
```

起動したコンテナの、OSとDartのバージョンを確認します。
``` bash
# cat /etc/lsb-release
DISTRIB_ID=Ubuntu
DISTRIB_RELEASE=14.04
DISTRIB_CODENAME=trusty
DISTRIB_DESCRIPTION="Ubuntu 14.04 LTS"
# dart --version
Dart VM version: 1.3.6 (Tue Apr 29 12:40:24 2014) on "linux_x64"
```

### まとめ
最近はNitrous.IOも使っていて、エディタやIDEへの依存を少なくしていますが、
やはり、このあといつもの開発用にVim,Emacsやマルチプレクサのインストールが必要になります。

Dockerfileに全部書くのが良いのか、[Salt States](http://docs.saltstack.com/en/latest/topics/tutorials/starting_states.html)を使った方がよいのか、
また悩み始めました。なんどか試行錯誤して自分にあった環境構築を探さないといけないです。

