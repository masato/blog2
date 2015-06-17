title: "DockerのHTTP Routing - Part3: OpenResty はじめに"
date: 2014-07-21 22:01:22
tags:
 - Docker
 - HTTPRouting
 - OpenResty
 - Lua
 - API開発
description: OpenRestyを使ったリバースプロキシを作ってみます。まずは慣れるために簡単なLuaプログラムを使り、Dockerコンテナの起動まで行ってみます。3scale/openrestyを使おうと思いましたが、API管理用途を想定しているようです。まだ理解が進んでいないので、もう少し簡単なDockerコンテナを使うことにします。
---

* `Update 2014-08-06`: [DockerのHTTP Routing - Part9: xip.io と OpenResty で動的リバースプロキシ](/2014/08/06/docker-reverse-proxy-xipio-openresty-sinatra/)
* `Update 2014-08-09`: [Dockerでデータ分析環境 - Part6: Ubuntu14.04のDockerにRStudio Serverと」OpenRestyをデプロイ](/2014/08/09/docker-analytic-sandbox-rstudio-openresty-deploy/)


[OpenResty](http://openresty.org/)を使ったリバースプロキシを作ってみます。
まずは慣れるために簡単なLuaプログラムを使ったDockerコンテナの起動まで行ってみます。

[3scale/openresty](https://github.com/3scale/docker-openresty)を使おうと思いましたが、READMEを読んでもよくわからなかったので、もう少し簡単なDockerコンテナを使うことにします。

<!-- more -->

### torhve/openresty-docker

[torhve/openresty-docker](https://github.com/torhve/openresty-docker)をcloneして使います。

READMEを読みながら使い方を確認します。

プロジェクトを作成します。

``` bash
$ mkdir ~/openresty_apps
$ cd !$
$ git clone https://github.com/torhve/openresty-docker.git
$ cd openresty-docker
$ mkdir logs
```

`Hello World!`を出力する簡単なLuaプログラムです。

``` lua ~/openresty_apps/openresty-docker/app.lua
ngx.say('Hello World!')
ngx.exit(200)
```

### イメージのビルドと実行

イメージをビルドします。

``` bash
$ docker build -t masato/openresty .
```

カレントディレクトリを`/helloproj`にマップして起動します。

``` bash
$ docker run --rm -t -i -p 8089:8080 -v `pwd`:/helloproj -w /helloproj masato/openresty 
```

### 確認

ブラウザを開いて確認します。

{% img center /2014/07/21/docker-reverse-proxy-openresty/openresty.png %}

### まとめ

OpenRestyと簡単なLuaプログラムの実行ができました。
次回はもう少しOpenRestyの構造に慣れてから、Dockerコンテナへリバースプロキシの設定をしてみます。

