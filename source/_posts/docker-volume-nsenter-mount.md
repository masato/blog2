title: "nsenterコマンドをDockerコンテナからホストにマウントして使う"
date: 2014-08-03 02:50:07
tags:
 - nsenter
 - Docker
description: Docker Online Meetup #3でメモした使い方です。これまでコンテナのデバッグ用にphusion/baseimageを使いsshdをインストールしていました。nsenterコマンドを使うとsshdをインストールしなくても起動したコンテナにattachすることができます。本当にsshdが必要なケースもあると思いますが、disposableな使い方を推進したいのでsshdはなるべく使わないようにしていきます。nsenterコマンドのインストールは、jpetazzo/nsenterのDockerコンテナからvolumeオプションを使い、コマンドをホストにマウントすることができます。
---

[Docker Online Meetup #3](/2014/07/31/docker-online-meetup-3/)でメモした使い方です。
これまでコンテナのデバッグ用に[phusion/baseimage](https://registry.hub.docker.com/u/phusion/baseimage/)を使いsshdをインストールしていました。

[nsenter](https://github.com/jpetazzo/nsenter)コマンドを使うとsshdをインストールしなくても起動したコンテナにattachすることができます。
本当にsshdが必要なケースもあると思いますが、disposableな使い方を推進したいのでなるべくsshdは使わないようにしていきます。

nsenterコマンドのインストールは、[jpetazzo/nsenter](https://registry.hub.docker.com/u/jpetazzo/nsenter/)のDockerコンテナからvolumeオプションを使い、コマンドをホストにマウントすることができます。

<!-- more -->

### jpetazzo/nsenter

[jpetazzo/nsenter](https://registry.hub.docker.com/u/jpetazzo/nsenter/)のDockerコンテナからnsenterコマンドをインストールします。

`--rm`オプションを使い、`jpetazzo/nsenter`を起動します。
`-v`オプションを使い、コンテナの`/target`をDockerホストの`/usr/local/bin`にマウントすることで、
nsenterコマンドをインストールします。

``` bash
$ sudo mkdir /usr/local/bin
$ docker run --rm -v /usr/local/bin:/target jpetazzo/nsenter
Installing nsenter to /target
Installing docker-enter to /target
```

### ~/bin/nse

nsenterコマンドを便利に使う`~/bin/nse`コマンドを作成します。

``` bash ~/bin/nse
#!/usr/bin/env bash
[ -n "$1" ] && sudo nsenter --pid --uts --mount --ipc --net --target $(docker inspect --format="&#123;&#123; .State.Pid &#125;&#125;" $1)
```

chmodで実行権限をつけます。

``` bash
$ chmod u+x ~/bin/nse
```

`CONTAINER_ID`を引数に指定して、コンテナにアタッチします。

``` bash
$ nse CONTAINER_ID
```

### 使い方

アタッチするコンテナは起動している必要があります。

``` bash
$ docker start b008d160f798
$ docker ps
CONTAINER ID        IMAGE                         COMMAND                CREATED             STATUS              PORTS                NAMES
b008d160f798        masato/nginx-minimal:latest   nginx -g 'daemon off   20 hours ago        Up 3 minutes        0.0.0.0:80->80/tcp   thirsty_poincare
```

`~/bin/nse`コマンドを使いコンテナにアタッチします。

``` bash
$ nse b008
root@b008d160f798:/#
```






