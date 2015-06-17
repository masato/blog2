title: "RStudio Serverの練習 - Part1: はじめに"
date: 2014-08-13 09:57:51
tags:
 - R
 - RStudioServer
 - runit
 - OpenResty
description: Dockerでデータ分析環境 - Part6 Ubuntu14.04のDockerにRStudio ServerとOpenRestyをデプロイで構築したDockerコンテナ上でRの練習をしていきます。実際にコードを書き始めると、いろいろ問題がでてきてDockerイメージの再作成をすることになりました。さっそく購入した現場ですぐ使える時系列データ分析を手を動かしながら読み進めていきます。
---

[Dockerでデータ分析環境 - Part6: Ubuntu14.04のDockerにRStudio ServerとOpenRestyをデプロイ](/2014/08/09/docker-analytic-sandbox-rstudio-openresty-deploy/)で構築したDockerコンテナ上でRの練習をしていきます。

実際にコードを書き始めると、いろいろ問題がでてきてDockerイメージの再作成をすることになりました。
さっそく購入した[現場ですぐ使える時系列データ分析](http://www.amazon.co.jp/dp/4774163015/)を手を動かしながら読み進めていきます。

<!-- more -->

### ファイルアップロードができない

イメージを再作成する前に、OpenRestyコンテナにnsenterで接続して直接修正してみます。

``` bash
$ docker-nsenter 51cad797dfaf
```

nginx.confを修正して、20MBまでファイルアップロード可能に設定します。

``` bash ~/docker_apps/openresty/nginx.conf
server {
...
        server_name  localhost;
        client_max_body_size 20M;
...
```

Supervisorのサービスを再起動します。

``` bash
# supervisorctl restart openresty
```

ローカルから2.2MBのファイルがアップロードできました。

{% img center /2014/08/13/rstudio-exercise-part1/rstudio-upload.png %}


### plotsが表示されない

右下のPlotsタブがあるパネルの表示が小さいとエラーになります。

``` bash
> attach(price4)
> plot(x5202,type="l",main="板硝子")
 以下にエラー plot.new() : figure margins too large
> 
```

パネルを最大にするか、グラフが表示できる程度に大きく表示するとエラーになりません。



### plotで文字化けする

こちらもDockerfileに戻る前に、RStudioコンテナにnsenterで接続して直接修正してみます。

``` bash
$ docker-nsenter 8f0d8fd9253c
```

日本語フォントをインストールして、runitのサービスを再起動します。

``` bash
# apt-get install fonts-takao
# sv restart rstudio-server
```

タイトルに日本語をつけてplotします。

``` bash
> plot(x5202,type="l",main="板硝子")
```

文字化けせずタイトルが日本語表示できました。

{% img center /2014/08/13/rstudio-exercise-part1/rstudio-plot.png %}

