title: "Snappy Ubuntu Core - Part1: VirtualBoxで動かす"
date: 2014-12-12 21:07:00
tags:
 - MicroOS
 - Ubuntu
 - SnappyUbuntuCore
 - qemu-img
 - VirtualBox
 - OVA
 - IDCFクラウド
description: 先日Snappy Ubuntu Coreが発表されました。CoreOSやProject Atomic、JeOSと同様にクラウドやコンテナ環境に適した、UbuntuのMicro OSバージョンです。現在はアルファバージョンで、Microsoft AzureとKVM用のイメージが公開されています。最近のAzureのLinuxへの対応はかなりアグレッシブです。手元にKVM環境がないので、イメージをVMDKに変換してVitualBoxで動かすことにします。
---

先日Snappy Ubuntu Coreが発表されました。CoreOSやProject Atomic、JeOSと同様にクラウドやコンテナ環境に適した、UbuntuのMicro OSバージョンです。現在はアルファバージョンで、Microsoft AzureとKVM用のイメージが公開されています。最近のAzureのLinuxへの対応はかなりアグレッシブです。手元にKVM環境がないので、イメージをVMDKに変換してVitualBoxで動かすことにします。

<!-- more -->

### Ubuntu 14.04の作業マシンの確認

IDCFクラウド上の作業マシンでVT-xが有効になっているか確認します。残念ながら有効になっていないので、この環境で直接KVMを動かすことはできないようです。

``` bash
$ sudo apt-get install qemu-kvm cpu-checker
$ kvm-ok
INFO: Your CPU does not support KVM extensions
KVM acceleration can NOT be used
```

[Announcing Snappy Ubuntu Core](http://www.ubuntu.com/cloud/tools/snappy)の手順を追いながら、VMDKファイルを使いローカルのVirtualBoxでSnappy Ubuntu Coreを動かしてみます。

### qemu-imgコマンドでVMDKにコンバート

作業マシンにqemu-utilsをインストールして、qemu-imgコマンドを使えるようにします。

``` bash
$ sudo apt-get update 
$ sudo apt-get install qemu-utils 
```

Ubuntu Coreのalpha releaseを作業マシンにダウンロードします。

``` bash
$ wget http://cdimage.ubuntu.com/ubuntu-core/preview/ubuntu-core-alpha-01.img
```

イメージフォーマットの確認をします。qcow2でした。

``` bash
$ qemu-img info ubuntu-core-alpha-01.img
image: ubuntu-core-alpha-01.img
file format: qcow2
virtual size: 20G (21474836480 bytes)
disk size: 108M
cluster_size: 65536
Format specific information:
    compat: 1.1
    lazy refcounts: false
```


qemu-imgコマンドを使い、qcow2からVMKD形式にコンバートします。

``` bash
$ qemu-img convert -O vmdk ubuntu-core-alpha-01.img ubuntu-core-alpha-01.vmdk
```


### OSXのVirtualBoxで起動する

Ubuntuの作業マシンでコンバートしたVMDKファイルをローカルのOSXにコピーしてきます。VirtualBox適当な仮想マシンを作成して起動します。

* RAM: 512MB
* ストレージ: ubuntu-core-alpha-01.vmdk

### OSXからSSH接続

OSXからVirtualBoxへSSH接続する場合、今回は簡単に仮想マシンのネットワークにポートフォワードを設定します。

* ネットワーク > アダプター1 > ポートフォワード > 2222 -> 22

OSXのターミナルからポートフォワードする2222へSSH接続します。

``` bash
$ ssh ubuntu@localhost -p 2222
```

デフォルトのユーザーとパスワードは以下です。

* user: ubuntu
* passwd: ubuntu


### インストールの確認

snappy infoコマンドを使うと、Ubuntu Coreのインストール状態をoverviewできます。まだframeworksもappsもインストールされていません。

``` bash
$ snappy info
release: ubuntu-core/devel
frameworks:
apps:
```

バージョンを確認すると、インストールしたubuntu-coreより新しい141が利用できるようです。

``` bash
$ snappy versions
Part         Tag   Installed  Available  Fingerprint     Active
ubuntu-core  edge  140        141        184ad1e863e947  *
```

apt-getコマンドは使えなくなっています。

``` bash
$ apt-get update
Ubuntu Core does not use apt-get, see 'snappy --help'!
$ sudo apt-get install docker
Ubuntu Core does not use apt-get, see 'snappy --help'!
```

### Docker(frameworks)のインストール

dockerパッケージを探します。1.3.2が見つかりました。

``` bash
$ snappy search docker
Part    Version    Description
docker  1.3.2.007  The docker app deployment mechanism
```

snappyコマンドを使い、Dockerをインストールします。

``` bash
$ sudo snappy install docker
docker      4 MB     [=============================================================]    OK
Part    Tag   Installed  Available  Fingerprint     Active
docker  edge  1.3.2.007  -          b1f2f85e77adab  *
```

インストールできたか確認します。

``` bash
$ snappy versions
Part         Tag   Installed  Available  Fingerprint     Active
ubuntu-core  edge  140        141        184ad1e863e947  *
docker       edge  1.3.2.007  -          b1f2f85e77adab  *
```

### hello-world(apps)のインストール

チュートリアルに従って、hello-worldアプリのインストールをしてみます。

``` bash
$ sudo snappy install hello-world
hello-world     31 kB     [========================================================]    OK
Part         Tag   Installed  Available  Fingerprint     Active
hello-world  edge  1.0.3      -          02a04059ae9304  *
```

snappy infoコマンドでインストール状態を確認します。

``` bash
$ snappy info
release: ubuntu-core/devel
frameworks: docker
apps: hello-world
```

snappyコマンドでインストールできるパッケージには2種類あります。はじめにインストールしたdockerはframeworks、hello-worldはappsに該当します。frameworksとappsはセキュリティポリシーが異なり、frameworksはシステムを拡張するためのパッケージのようです。dockerパッケージはdockerアプリを動かすためのサービスになります。


### webserver(apps)をインストール

もう少し複雑になったwebserverアプリを検索してみます。

``` bash
$ snappy search webserver
Part                  Version  Description
go-example-webserver  1.0.1    Minimal Golang webserver for snappy
xkcd-webserver        0.3.1    Show random XKCD compic via a build-in webserver
```

go-example-webserverはGoで書かれたWebサーバーです。

``` bash
$ sudo snappy install go-example-webserver
go-example-webserver      1 MB     [===============================================]    OK
Part                  Tag   Installed  Available  Fingerprint     Active
go-example-webserver  edge  1.0.1      -          444d0831d25718  *
```

VirtualBoxの仮想マシンのネットワークに、8888をポートフォワードを追加します。OSXのターミナルからcurlで接続するとHello Worldが出職できました。

``` bash
$ curl localhost:8888
Hello World
```

### バージョンアップ

新しい更新のバージョンがあるか確認します。

``` bash
$ snappy update-versions
1 updated components are available with your current stability settings.
$ snappy versions
Part                  Tag   Installed  Available  Fingerprint     Active
ubuntu-core           edge  140        141        184ad1e863e947  *
docker                edge  1.3.2.007  -          b1f2f85e77adab  *
go-example-webserver  edge  1.0.1      -          444d0831d25718  *
hello-world           edge  1.0.3      -          02a04059ae9304  *
```

ubuntu-coreパッケージのupdateを実行します。
 
``` bash
$ sudo snappy update ubuntu-core
ubuntu-core     61 MB     [======================================================]    OK
Reboot to use the new ubuntu-core.
```

-aフラグをつけて、アクティブでないバージョンも表示すると、ubuntu-coreのバージョン141がインストールされました。最終行に出力されるようにリブート後に最新のバージョンにアップデートされます。
 
``` bash
$ snappy versions -a
Part                  Tag   Installed  Available  Fingerprint     Active
ubuntu-core           edge  140        141        184ad1e863e947  *
ubuntu-core           edge  141        -          7f068cb4fa876c  R
docker                edge  1.3.2.007  -          b1f2f85e77adab  *
go-example-webserver  edge  1.0.1      -          444d0831d25718  *
hello-world           edge  1.0.3      -          02a04059ae9304  *
Reboot to use the new ubuntu-core.
```

OSをリブートします。

``` bash
$ sudo reboot
```

バージョンを確認します。ubuntu-coreが141に更新されました。
 
``` bash
$ snappy versions -a
Part                  Tag   Installed  Available  Fingerprint     Active
ubuntu-core           edge  141        -          7f068cb4fa876c  *
ubuntu-core           edge  140        -          184ad1e863e947  R
docker                edge  1.3.2.007  -          b1f2f85e77adab  *
go-example-webserver  edge  1.0.1      -          444d0831d25718  *
hello-world           edge  1.0.3      -          02a04059ae9304  *
```

### アップデートをロールバックする

Snappy Ubuntu Coreは簡単にシステムのロールバックができます。ubuntu-coreをバージョン140にrollbackしてみます。
 
``` bash
$ sudo snappy rollback ubuntu-core
Rolling back ubuntu-core: (edge 141 7f068cb4fa876c -> edge 140 184ad1e863e947)
Reboot to use the new ubuntu-core.
$ sudo reboot
```

OSをリブートすると、ubuntu-coreがバージョン140にrollbackしたことを確認できます。この後はOSをリブートしただけだとバージョンは変更されません。

``` bash
Part                  Tag   Installed  Available  Fingerprint     Active
ubuntu-core           edge  140        141        184ad1e863e947  *
ubuntu-core           edge  141        -          7f068cb4fa876c  R
docker                edge  1.3.2.007  -          b1f2f85e77adab  *
go-example-webserver  edge  1.0.1      -          444d0831d25718  *
hello-world           edge  1.0.3      -          02a04059ae9304  *
Reboot to use the new ubuntu-core.
```

### OVA形式でエクスポートしてみる

OSXのVirtualBoxからイメージを単純にOVA形式でエクスポートして、IDCFクラウドにインポートしてみました。残念ながらデフォルトの状態だとインポートに失敗してしまいます。OVFファイルの中身を編集しながらOVAを作ってみようと思います。
