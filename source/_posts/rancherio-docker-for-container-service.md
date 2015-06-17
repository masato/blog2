title: "Rancher.ioでDockerコンテナを管理する - Part1: はじめに"
date: 2014-12-14 15:57:44
tags:
 - Docker管理
 - Rancherio
 - Cattle
 - Stampedeio
description: Rancher.ioというオープンソースのDockerコンテナ管理ツールがあります。開発元のRancher Labsのホームページによると、特定のインフラやIaaSに依存しない、Dockerコンテナ管理とSoftware Defined Infrastructureを組み合わせたプラットフォームを構築するソフトウェアを目指すようです。At Rancher, we’re developing software that will allow anyone to build their own Docker container service on any infrastructure or cloud. You can follow our open source project at Rancher.io.
---

[Rancher.io](https://github.com/rancherio/rancher)というオープンソースのDockerコンテナ管理ツールがあります。開発元の[Rancher Labs](http://www.rancher.com/)のホームページによると、特定のインフラやIaaSに依存しない、Dockerコンテナ管理とSoftware Defined Infrastructureを組み合わせたプラットフォームを構築するソフトウェアを目指すようです。

> At Rancher, we’re developing software that will allow anyone to build their own Docker container service on any infrastructure or cloud. You can follow our open source project at Rancher.io.

<!-- more -->

### Stampede.io 

2014年8月にCitrixに在籍していたDarren Shepherdが[Announcing Stampede.io: A Hybrid IaaS/Docker Orchestation Platform Running on CoreOS](http://www.ibuildthecloud.com/blog/2014/08/21/announcing-stampede-dot-io-a-hybrid-iaas-slash-docker-orchestation-platform-running-on-coreos/)というブログで、[Stampede.io](https://github.com/cattleio/stampede)を公開しました。Digital Ocean上のCoreOSに12万コンテナを作成したプレゼン資料が[SlideShare](http://www.slideshare.net/DarrenShepherd1/stampedeio-coreos-digital-ocean-meetup)にあります。

### Rancher Labs

その後Stampede.ioは[Rancher.io](https://github.com/rancherio/rancher)となり、元Citrixのメンバーが起業した[Rancher Labs](http://www.rancher.io/)に開発が移行しました。Citrixに買収される前のCloud.comを起業したSheng LiangがCEOを勤め、Shannon Williams、Darren Shepherdと、CloudStackを開発していたメンバーが[共同創設者](http://www.rancher.com/our-team/)になっています。チーフアーキテクトのDarren ShepherdはCloudStackとGoddayのアーキテクトを勤めていました。

### Rancher.io

Stampede.ioはCoreOSとfleetによるアークテクチャでしたが、Rancher.ioではDockerホストにUbuntu14.04も使えるようになっています。Rancher.ioはどのIaaSでも同じようにDockerを動かせるインフラを実現するために、Docker管理ツールの提供とSoftware Definedなストレージとネットワークの[実装を計画](https://github.com/rancherio/rancher)しています。CloudStackがハイパーバイザーを抽象化したように、Rancher.ioはIaaSを抽象化してくれるようです。
 
### Cattle

Stampede.ioと同様にRancher.ioもIaaSのオーケストレーションプラットフォームとして、JavaとSpringで書かれた[Cattle](https://github.com/rancherio/cattle)を使っています。実際にRancher.ioのソースコードを読み、手を動かしながらアーキテクチャを理解していこうと思います。インストールの手順を読むと、Management ServerとDocker Nodesによって構成されるようです。絶賛開発中のため今後大きな変更もありそうですが現状の理解からはじめます。

### Management Server (Cattle)

Management Serverの[Dockerfile](https://github.com/rancherio/rancher/blob/master/server/Dockerfile)を読むと、ビルドした[Cattle](https://github.com/rancherio/cattle)のJARファイルをデプロイしています。[起動スクリプト](https://github.com/rancherio/rancher/blob/master/server/artifacts/cattle.sh)によると、MySQLやZooKeeperのセットアップも読めますが今のところ使っていません。Cattleのデータベースは`/var/lib/cattle/database`にあるH2のようです。


### Docker Nodes (python-agent)

Docker Nodesの方は、もちろんDockerと、docker-py、[python-agent](https://github.com/rancherio/python-agent)をインストールします。pyagentはcAdvisorやコンソールのためのWebSocketサーバーの役割もします。Agentは今のところRancher.ioの管理画面を開くブラウザから、WebSocketで接続可能なIPアドレスを指定する必要があります。

### Rancher.ioのインストール方法

Rancher.ioのインストール方法は3つあります。

* インストールはDockerホストを数台作成し、手動で管理用のDockerコンテナを作成する方法
* [Vagrant](https://github.com/rancherio/rancher/blob/master/Vagrantfile)を使う方法
* [Puppet](https://github.com/nickschuch/puppet-rancher)を使う方法

全体を理解するために手動でDockerコンテナを起動して確認してみます。
