title: "Panamax in CoreOS on IDCFクラウド - Part1: インストール"
date: 2014-08-19 00:31:33
tags:
 - Docker管理
 - Panamax
 - CoreOS
 - Deis
 - IDCFクラウド
description: ISOからインストールして用意した367.1.0のCoreOSにさっそくPanamaxをインストールしてみます。Panamaxは、DockerやCoreOSの勉強でお世話になっている、CenturyLinkLabsがオープンソースで公開しているDockerの管理ツールです。The New Stack Analysts Docker’s Future is in the Orchestrationのインタビューの中で、CenturyLinkLabsのLucas Carlson氏が以下のように応えています。There’s a big difference between a hello world, approach and more sophisticated usage. CenturyLinkLabsがDockerの教育に力を入れているのはすばらしいことです。特にCoreOSやFigに対して注目が高いので、Panamaxにも期待しています。
---

[ISOからインストールして](/2014/08/18/idcf-coreos-install-from-iso/)用意した367.1.0のCoreOSにさっそくPanamaxをインストールしてみます。

Panamaxは、DockerやCoreOSの勉強でお世話になっている、CenturyLinkLabsがオープンソースで公開しているDockerの管理ツールです。

[The New Stack Analysts: Docker’s Future is in the Orchestration](http://thenewstack.io/the-new-stack-analysts-dockers-future-is-in-the-orchestration/)のインタビューの中で、CenturyLinkLabsのLucas Carlson氏が以下のように応えています。
> There’s a big difference between a ‘hello world,’ approach and more sophisticated usage. 

CenturyLinkLabsがDockerの教育に力を入れているのはすばらしいことです。特にCoreOSやFigに対して注目が高いので、Panamaxにも期待しています。

<!-- more -->

### Panamaxのインストール

CoreOSのインスタンスにSSH接続します。

``` bash
$ eval `ssh-agent`
$ ssh-add ~/.ssh/deis
$ ssh -A core@10.1.0.66
```

CoreOSのバージョンを確認します。

``` bash
$ cat /etc/os-release
NAME=CoreOS
ID=coreos
VERSION=367.1.0
VERSION_ID=367.1.0
BUILD_ID=
PRETTY_NAME="CoreOS 367.1.0"
ANSI_COLOR="1;32"
HOME_URL="https://coreos.com/"
BUG_REPORT_URL="https://github.com/coreos/bugs/issues"
```

インストーラーをダウンロードして実行します。Panamaxのバージョンは`0.1.2`です。

``` bash
$ sudo -i
$ mkdir panamax && cd panamax
$ wget http://download.panamax.io/installer/pmx-installer-latest.zip
$ unzip pmx-installer-latest.zip
Archive:  pmx-installer-latest.zip
  inflating: create-docker-mount
  inflating: LICENSE
  inflating: desktop
 extracting: panamax
  inflating: .coreosenv
  inflating: README.md
  inflating: ubuntu.sh
  inflating: Vagrantfile
 extracting: .version
  inflating: coreos
  inflating: panamax.rb
$ cat .version
0.1.2
$ ./coreos install --stable
Installing Panamax...

docker pull centurylink/panamax-api:latest
..
docker pull centurylink/panamax-ui:latest
..

docker pull google/cadvisor:0.1.0
..
Panamax install complete
```

### 確認

ブラウザから確認します。3000ポートでLISTENしています。

Imagesのパネルには、Panamaxのインストールで`docker pull`した上記のイメージが3つ表示されています。

{% img center /2014/08/19/panamax-in-coreos-on-idcf/panamax-dashboard.png %}

### まとめ

とりあえずPanamaxをインストールしてみました。
たくさんテンプレートが用意されていて、またDockerインデックスからも直接イメージの検索と実行ができるようです。
fleet用のsystemdのunitを書くのがちょっと面倒なときに、画面からデプロイができるツールは便利です。

Deisのアプリで使うPostgreSQLのイメージをPanamaxからデプロイして管理してみようと思います。



