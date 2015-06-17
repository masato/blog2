title: 'Couchbase on Docker - Part4: CoreOS自動アップデート後のRebalance'
date: 2015-01-22 20:10:55
tags:
 - CoreOS
 - Couchbase
 - CouchbaseServer
 - IDCFクラウド
 - fleetctl
description: CoreOSクラスタ上に構築したCouchbase Serverクラスタをしばらく放置していたらいくつかunitsが死んでいました。CoreOSのreboot-strategyはデフォルトのbest-effortです。CoreOSの再起動後に起動に失敗しているfleet unitsがありました。
---


CoreOSクラスタ上に構築したCouchbase Serverクラスタをしばらく放置していたらいくつかunitsが死んでいました。CoreOSのreboot-strategyはデフォルトのbest-effortです。CoreOSの再起動後に起動に失敗しているfleet unitsがありました。

<!-- more -->

## fleet unitsの確認

fleetctlのバージョンは0.8.3です。

``` bash
$ fleetctl version
fleetctl version 0.8.3
```

fleetctlコマンドでunitsをリストします。Couchbase Serverは3ノード中2ノードがactiveなので、動作はしている状態です。

``` bash
$ fleetctl list-units
UNIT						MACHINE			ACTIVE		SUB
couchbase_bootstrap_node.service		f0f5fcc4.../10.3.0.47	active		running
couchbase_bootstrap_node_announce.service	f0f5fcc4.../10.3.0.47	active		running
couchbase_node.1.service			33bf6324.../10.3.0.7	inactive	dead
couchbase_node.2.service			461b77ab.../10.3.0.193	active		running
sync_gw_announce@1.service			33bf6324.../10.3.0.7	inactive	dead
sync_gw_node@1.service				33bf6324.../10.3.0.7	inactive	dead
```

## CorOSのバージョンを確認

現在のCoreOSクラスタの構成は以下の通りです。

``` bash
$ fleetctl list-machines
MACHINE		IP		METADATA
33bf6324...	10.3.0.7	-
461b77ab...	10.3.0.193	-
80e8c0f2...	10.3.0.210	-
f0f5fcc4...	10.3.0.47	-
```

最初CoreOSのバージョンは`494.5.0`をインストールしていましたが`522.4.0`に自動アップデートされています。

``` bash
$ fleetctl ssh 33bf6324 'cat /etc/os-release | grep VERSION_ID'
VERSION_ID=522.4.0
$ fleetctl ssh 461b77ab 'cat /etc/os-release | grep VERSION_ID'
VERSION_ID=522.5.0
$ fleetctl ssh 80e8c0f2 'cat /etc/os-release | grep VERSION_ID'
VERSION_ID=522.4.0
$ fleetctl ssh f0f5fcc4 'cat /etc/os-release | grep VERSION_ID'
VERSION_ID=522.4.0
```

## 停止しているunitsの再起動

dead状態になっているfleet unitsをloadしてlaunchさせます。

``` bash
$ fleetctl stop couchbase_node.1.service
Unit couchbase_node.1.service loaded on 33bf6324.../10.3.0.7
$ fleetctl start couchbase_node.1.service
Unit couchbase_node.1.service launched on 33bf6324.../10.3.0.7
$ fleetctl stop sync_gw_node@1.service
Unit sync_gw_node@1.service loaded on 33bf6324.../10.3.0.7
$ fleetctl start sync_gw_node@1.service
Unit sync_gw_node@1.service launched on 33bf6324.../10.3.0.7
$ fleetctl stop sync_gw_announce@1.service
Unit sync_gw_announce@1.service loaded on 33bf6324.../10.3.0.7
c$ fleetctl start sync_gw_announce@1.service
Unit sync_gw_announce@1.service launched on 33bf6324.../10.3.0.7
```

すべてのunitsが正常に動作しました。

```
$ fleetctl list-units
UNIT						MACHINE			ACTIVE	SUB
couchbase_bootstrap_node.service		f0f5fcc4.../10.3.0.47	active	running
couchbase_bootstrap_node_announce.service	f0f5fcc4.../10.3.0.47	active	running
couchbase_node.1.service			33bf6324.../10.3.0.7	active	running
couchbase_node.2.service			461b77ab.../10.3.0.193	active	running
sync_gw_announce@1.service			33bf6324.../10.3.0.7	active	running
sync_gw_node@1.service				33bf6324.../10.3.0.7	active	running
```

## Server NodesのRebalance

Couchbase管理Webコンソールにログインして、Server NodesタブからRebalanceボタンをクリックします。Couchbaseは画面から簡単にクラスタのノード管理ができます。

![rebalance.png](/2015/01/22/idcf-coreos-couchbase-update-rebase/rebase.png)

