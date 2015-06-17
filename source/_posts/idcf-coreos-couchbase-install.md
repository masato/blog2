title: 'Couchbase on Docker - Part1: Couchbase Server 3.0.1クラスタをインストール'
date: 2014-12-20 12:34:28
tags:
 - CoreOS
 - Couchbase
 - CouchbaseServer
 - IDCFクラウド
 - fleetctl
description: 前回IDCFクラウドにCoreOS 494.5.0をISOからインスト-ルしました。同じ方法でISOから同じdiscoveryのURLを指定して、3台のCoreOSクラスタを構築します。Running Couchbase Cluster Under CoreOS on AWSはAWS CloudFormationの例ですが、CoreOSクラスタを用意すれば同じfleet unit filesを使いCouchbase Server 3.0.1クラスタをインストールできます。ここがcloud provider agnosticなCoreOSのよいところです。
---

前回IDCFクラウドに[CoreOS 494.5.0をISOからインスト-ル](/2014/12/19/idcf-coreos-install-from-iso-49450/)しました。同じ方法でISOから同じdiscoveryのURLを指定して、3台のCoreOSクラスタを構築します。[Running Couchbase Cluster Under CoreOS on AWS](http://tleyden.github.io/blog/2014/11/01/running-couchbase-cluster-under-coreos-on-aws/)はAWS CloudFormationをの例ですが、CoreOSクラスタを用意すれば同じ[fleet unit files](https://github.com/couchbaselabs/couchbase-server-docker/tree/master/3.0.1/fleet)を使いCouchbase Server 3.0.1クラスタをインストールできます。ここがcloud provider agnosticなCoreOSのよいところです。

<!-- more -->

### Couchbase Server

Couchbase Serverは分散型NoSQLドキュメント指向データベースです。REST APIを使ってJSONの操作ができます。Couchbaseは歴史上おもしろい特徴を持っています。CouchDBから始まるドキュメント型データベースの側面と、Membaseとの統合によるmemcached互換の分散メモリキャッシュサーバの側面があります。

### couchbaselabs/couchbase-server-docker/3.0.1

[Couchbase Labs](http://labs.couchbase.com/)プロジェクトがGitHubに公開しているインストーラーを使います。

* [couchbaselabs/couchbase-server-docker/3.0,1](https://github.com/couchbaselabs/couchbase-server-docker/tree/master/3.0.1)

Docker Hub Registryです。

* [tleyden5iwx/couchbase-server-3.0.1](https://registry.hub.docker.com/u/tleyden5iwx/couchbase-server-3.0.1/)


Docker Hub Registryにあるアーキテクチャ図を転載します。IDCFクラウドの場合AWS CloudFormationの代わりにfleetでCoreOSクラスタを構築します。

{% img center /2014/12/20/idcf-coreos-couchbase-install/couchbase-coreos-onion.png %}


### CoreOSクラスタの確認

前回IDCFクラウドに3台構成のCoreOSクラスタを構築しました。

``` bash
$ fleetctl list-machines
MACHINE         IP              METADATA
461b77ab...     10.3.0.193      -
80e8c0f2...     10.3.0.210      -
f0f5fcc4...     10.3.0.7        -
```

### cluster-init.shスクリプト

cluster-init.shスクリプトをダインロードして実行権限をつけます。

``` bash
$ wget https://raw.githubusercontent.com/couchbaselabs/couchbase-server-docker/master/scripts/cluster-init.sh
$ chmod +x cluster-init.sh
```

今回指定するcluster-init.shのオプションは以下の通りです。


* -v: バージョン -> `3.0.1`
* -n: Couchbaseのノード数 -> `3`
* -u: ユーザー名:パスワード -> `user:passw0rd`

``` bash cluster-init.sh
#!/bin/sh
# Usage:
#
# ./cluster-init.sh -n 3 -u "user:passw0rd"
#
# Where:
#   -v Couchbase Server version (3.0.1 or 2.2)
#   -n number of couchbase nodes to start
#   -u the username and password as a single string, delimited by a colon (:)
# 
# This will start a 3-node couchbase cluster (so you will need to have kicked off
# a cluster with at least 3 ec2 instances)
usage="./cluster-init.sh -v 3.0.1 -n 3 -u \"user:passw0rd\""
```

インストールスクリプトを実行します。

``` bash
$ ./cluster-init.sh -v 3.0.1 -n 3 -u "user:passw0rd"
Cloning into 'couchbase-server-docker'...
remote: Counting objects: 333, done.
remote: Compressing objects: 100% (15/15), done.
remote: Total 333 (delta 6), reused 0 (delta 0)
Receiving objects: 100% (333/333), 39.22 KiB | 0 bytes/s, done.
Resolving deltas: 100% (174/174), done.
Checking connectivity... done.
user:passw0rd
Unit couchbase_node.2.service launched on f0f5fcc4.../10.3.0.7
Unit couchbase_bootstrap_node.service launched on 461b77ab.../10.3.0.193
Unit couchbase_bootstrap_node_announce.service launched on 461b77ab.../10.3.0.193
Unit couchbase_node.1.service launched on 80e8c0f2.../10.3.0.210
```

ログに出力されているように、インストールスクリプトは以下の処理を行います。

* fleet unit filesをダウンロード
* 指定したノード数のfleet unit filesをテンプレートから生成
* etcdにCouchbaseのユーザー名とパスワードをセット
* fleetctlでservicesをインストール


### Couchbaseクラスタの確認

fleetctlを使いCouchbase Serverのunitsが正常に動作していることを確認します。

``` bash
$ fleetctl list-units
UNIT                                            MACHINE                 ACTIVE  SUB
couchbase_bootstrap_node.service                461b77ab.../10.3.0.193  active  running
couchbase_bootstrap_node_announce.service       461b77ab.../10.3.0.193  active  running
couchbase_node.1.service                        80e8c0f2.../10.3.0.210  active  running
couchbase_node.2.service                        f0f5fcc4.../10.3.0.7    active  running
```

Couchbase Serverのnodeは、couchbase_bootstrap_nodeを含めて3台作成できました。

* couchbase_bootstrap_node.service
* couchbase_node.1.service
* couchbase_node.2.service


### Couchbase Server Web Admin

ブラウザを開き、Couchbase ServerのCoreOSのどれか1台に接続します。UsernameとPasswordは、cluster-init.shスクリプトの-uフラグで指定した、`user:passw0rd`です。
http://10.3.0.193:8091
`fleetctl list-units`で表示しているのはプライベートIPアドレスです。プライベートIPアドレスに直接ブラウザで接続できない場合は、パブリックIPアドレスをどれか1つのCoreOS仮装マシンにポートフォワードします。

{% img center /2014/12/20/idcf-coreos-couchbase-install/couchbase-console.png %}


また、Dockerコンテナのlocaleを日本にしておくとログの日付が見やすくなります。

``` bash
$ docker exec -it ff15f56166b7 bash
$ ln -sf ../usr/share/zoneinfo/Japan /etc/localtime
```

### Couchbaseクラスタをリバランスする

ブラウザでCouchbase Server Web Adminを開くと、Pending Rebalanceに2つのノードが残っているので、最初のリバランスをします。

* Server Nodes -> Rebalance

{% img center /2014/12/20/idcf-coreos-couchbase-install/couchbase-rebalance.png %}

