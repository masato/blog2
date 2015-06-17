title: 'IDCFクラウドにDockerをインストール'
date: 2014-05-17 21:33:40
tags:
 - IDCFクラウド
 - Docker
 - Vagrant
description: この前GCEにDockerをインストールしました。今回は普段使っているIDCFクラウドにもDockerをインストールしてみようと思います。
---
[この前](/2014/05/13/gce-docker/)GCEにDockerをインストールしました。
今回は普段使っているIDCFクラウドにもDockerをインストールしてみようと思います。

<!-- more -->

### 準備

Dockerをインストールするための準備をします。

``` bash
$ uname -a
Linux matedesk 3.11.0-12-generic #19-Ubuntu SMP Wed Oct 9 16:20:46 UTC 2013 x86_64 x86_64 x86_64 GNU/Linux
```

必要なパッケージをインストールします。

``` bash
$ sudo apt-get update
$ sudo apt-get install linux-image-extra-`uname -r`
```

### Dockerのインストール

インストーラーの実行

``` bash
$ curl -s https://get.docker.io/ubuntu/ | sudo sh
```

バージョンの確認

``` bash
$ docker version
Client version: 0.11.1
Client API version: 1.11
Go version (client): go1.2.1
Git commit (client): fb99f99
Server version: 0.11.1
Server API version: 1.11
Git commit (server): fb99f99
Go version (server): go1.2.1
Last stable version: 0.11.1
```

sudo なしでdockerコマンドを使えるようにします。 dockerグループに追加したらログインし直します。

``` bash
$ sudo usermod -aG docker $USER
$ exit
```

### Dockerの実行

Dockerがインストールされたかどうか確認します。

``` bash
$ docker run -i -t ubuntu:12.04 /bin/bash
Unable to find image 'ubuntu:12.04' locally
Pulling repository ubuntu
74fe38d11401: Download complete
511136ea3c5a: Download complete
f10ebce2c0e1: Download complete
82cdea7ab5b5: Download complete
5dbd9cb5a02f: Download complete
root@4b458e6e2946:/# exit
```

docker psで停止を確認します。

``` bash
$ docker ps -a
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS                     PORTS               NAMES
4b458e6e2946        ubuntu:12.04        /bin/bash           23 seconds ago      Exited (0) 6 seconds ago                       tender_curie
```

### まとめ

簡単にDockerはインストールできました。大きめのインスタンスを作成してDockerを利用するとリソース活用できます。
これは[この前](/2014/05/09/vagrant-on-idcf-64bit-lxc/)Vagrant経由でLXCを使った場合と同様です。
また、Vagrant1.6で`Docker Provider`が追加されました。次回は、VagrantのProviderとしてDockerを試してみようと思います。
