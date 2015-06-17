title: 'Dockerで開発環境をつくる - puppies vs. cows'
date: 2014-05-22 22:37:31
tags:
 - Dockerfile
 - DockerDevEnv
 - Emacs
 - Ubuntu
 - Puppies-vs-Cattle
 - PistonCloud
description: 去年PistonCloudからTシャツをもらったのを思い出しました。コンテナをしばらく使っていると、日本語設定を入れるのが面倒なので同じコンテナで複数アプリを開発するようになりました。disposableは遠いです。やり直しで日本語用にDockerfileを書くようにします。とりあえず、Ubuntu14.04の日本語環境の構築と、Emacs24のインストールまで行います。やはりSSHサーバーは必要みたいです。
---

去年PistonCloudから[Tシャツ](http://www.networkworld.com/community/blog/puppies-or-cows)をもらったのを思い出しました。
コンテナをしばらく使っていると、日本語設定を入れるのが面倒なので同じコンテナで複数アプリを開発するようになりました。
disposableは遠いです。

やり直しで日本語用にDockerfileを書くようにします。
とりあえず、Ubuntu14.04の日本語環境の構築と、Emacs24のインストールまで行います。
やはりSSHサーバーは必要みたいです。

<!-- more -->

### Dockerfileの作成

[passenger-docker](https://github.com/phusion/baseimage-docker)を参考にして、Dockerfileを書きます。
コメントを読むとバージョンを固定して使うみたいです。[Changelog](https://github.com/phusion/baseimage-docker/blob/master/Changelog.md)をみて、Dockerfileは作り直していきます。

プロジェクトの作成
``` bash
$ mkdir -p ~/docker_apps/phusion
$ cd !$
```

コンテナへコピーする公開鍵とドットファイルは予め用意しておきます。

``` Dockerfile
FROM phusion/baseimage:0.9.10

# Set correct environment variables.
ENV HOME /root

# Regenerate SSH host keys.
RUN /etc/my_init.d/00_regen_ssh_host_keys.sh

## Install an SSH of your choice.
ADD google_compute_engine.pub /tmp/your_key
RUN cat /tmp/your_key >> /root/.ssh/authorized_keys && rm -f /tmp/your_key

## apt-get update
RUN apt-get -yq update

## Japanese Environment
RUN apt-get install -y language-pack-ja
ENV LANG ja_JP.UTF-8
RUN update-locale LANG=ja_JP.UTF-8

## Development Environment
RUN apt-get install -y emacs24-nox emacs24-el

## dotfiles
RUN mkdir .emacs.d
ADD dotfiles/.emacs.d/init.el /root/.emacs.d/init.el

CMD ["/sbin/my_init"]

RUN apt-get clean && rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*
```

### イメージの作成
Dockerfileからイメージをつくります。 phusionにならって、baseimageにします。
``` bash
$ docker build -t masato/baseimage .
```

### コンテナの起動

作成したイメージを起動して確認します。`--rm`オプションでexitしたらコンテナは削除します。
``` bash
$ docker run --rm -t -i masato/baseimage /sbin/my_init -- bash -l
*** Running /etc/my_init.d/00_regen_ssh_host_keys.sh...
*** Running /etc/rc.local...
*** Booting runit daemon...
*** Runit started as PID 15
*** Running bash -l...
root@aa8f76299e8c:/#
```

`locale`を確認するとja_JP.UTF-8になっています。

``` bash
root@aa8f76299e8c:/# locale
LANG=ja_JP.UTF-8
LANGUAGE=
LC_CTYPE="ja_JP.UTF-8"
LC_NUMERIC="ja_JP.UTF-8"
LC_TIME="ja_JP.UTF-8"
LC_COLLATE="ja_JP.UTF-8"
LC_MONETARY="ja_JP.UTF-8"
LC_MESSAGES="ja_JP.UTF-8"
LC_PAPER="ja_JP.UTF-8"
LC_NAME="ja_JP.UTF-8"
LC_ADDRESS="ja_JP.UTF-8"
LC_TELEPHONE="ja_JP.UTF-8"
LC_MEASUREMENT="ja_JP.UTF-8"
LC_IDENTIFICATION="ja_JP.UTF-8"
LC_ALL=
```

### コンテナへの接続

シェルを起動する場合。
``` bash
$ docker run -t -i masato/baseimage /sbin/my_init /bin/bash
*** Running /etc/my_init.d/00_regen_ssh_host_keys.sh...
*** Running /etc/rc.local...
*** Booting runit daemon...
*** Runit started as PID 15
*** Running /bin/bash...
root@3c19c33ac5dc:/#
```

コンテナの起動後にSSHで接続する場合。
```
$ docker inspect 3c19c33ac5dc | jq -r '.[0] | .NetworkSettings | .IPAddress'
172.17.0.2
$ ssh root@172.17.0.2 -i ./google_compute_engine
```

### まとめ
とりあえずDockerfileの作り直しました。コンテナは一度起動したら、rebootもしない、システムに変更は加えないように作るようにします。
起動が遅くなるのでなるべくapt-getしないで、最低限にしたいです。しばらくDockerfileの再作成は頻繁になりそうです。






