title: "CoreOSをfleetctlから操作する - Part2: hello.service"
date: 2014-07-26 18:49:30
tags:
 - CoreOS
 - fleet
 - fleetctl
 - etcdctl
description: Part1と同様にIDCFクラウドにCoreOSクラスタを構築するで構築したCoreOSクラスタを使い、fleetctlの使い方を勉強していきます。はじめてのUnitはExperimenting with CoreOS, confd, etcd, fleet, and CloudFormationを参考にして、hello.serviceを登録してみます。
---

[Part1](/2014/07/25/coreos-fleetctl-endpoint-tunnell-options/)と同様に[IDCFクラウドにCoreOSクラスタを構築する - Part5: 再セットアップ](/2014/07/19/idcf-coreos-cluster-re-setting-up/)で構築したCoreOSクラスタを使い、fleetctlの使い方を勉強していきます。

はじめてのUnitは[Experimenting with CoreOS, confd, etcd, fleet, and CloudFormation](http://marceldegraaf.net/2014/04/24/experimenting-with-coreos-confd-etcd-fleet-and-cloudformation.html)を参考にして、hello.serviceを登録してみます。

<!-- more -->

### はじめてのUnit - hello.service

プロジェクトを作成します。

``` bash
$ mkdir -p ~/fleetctl_apps/hello
$ cd !$
```

登録するUnitを記述したserviceファイルです。

``` bash ~/fleetctl_apps/hello/hello.service
[Unit]
Description=Hello World
After=docker.service
Requires=docker.service
 
[Service]
EnvironmentFile=/etc/environment
ExecStartPre=/usr/bin/etcdctl set /test/%m ${COREOS_PRIVATE_IPV4}
ExecStart=/usr/bin/docker run --name test --rm busybox /bin/sh -c "while true; do echo Hello World; sleep 1; done"
ExecStop=/usr/bin/etcdctl rm /test/%m
ExecStop=/usr/bin/docker kill test
```

### hello.serviceをUnitに登録

hello.serviceをCoreOSクラスタのUnitに登録します。

``` bash
$ export FLEETCTL_ENDPOINT=http://10.1.0.174:4001
$ fleetctl submit hello.service
$ fleetctl start hello.service
Job hello.service launched on 3b9bf346.../10.1.1.90
```

### Unitの確認

登録したUnitをcatで表示します。

``` bash ~/fleetctl_apps/hello/hello.service
[Unit]
Description=Hello World
After=docker.service
Requires=docker.service

[Service]
EnvironmentFile=/etc/environment
ExecStartPre=/usr/bin/etcdctl set /test/%m ${COREOS_PRIVATE_IPV4}
ExecStart=/usr/bin/docker run --name test --rm busybox /bin/sh -c "while true; do echo Hello World; sleep 1; done"
ExecStop=/usr/bin/etcdctl rm /test/%m
ExecStop=/usr/bin/docker kill test
```

list-unitsコマンドで、Unitの一覧を表示します。

``` bash
$ fleetctl list-units
UNIT            DSTATE          TMACHINE                STATE           MACHINE                 ACTIVE
hello.service   launched        3b9bf346.../10.1.1.90   launched        3b9bf346.../10.1.1.90   active
```

journalコマンドで、echoが確認できます。Unitは正常に動作しているようです。

``` bash
$ fleetctl journal hello.service
Jul 26 11:04:08 coreos-beta-v3-2 docker[6802]: Hello World
Jul 26 11:04:09 coreos-beta-v3-2 docker[6802]: Hello World
Jul 26 11:04:10 coreos-beta-v3-2 docker[6802]: Hello World
Jul 26 11:04:11 coreos-beta-v3-2 docker[6802]: Hello World
```

statusコマンドで、Unitの状態を確認できます。

``` bash
$ fleetctl status hello.service
fleetctl status hello.service
● hello.service - Hello World
   Loaded: loaded (/run/fleet/units/hello.service; linked-runtime)
   Active: active (running) since Mon 2014-07-26 11:01:39 UTC; 4min 31s ago
  Process: 6794 ExecStartPre=/usr/bin/etcdctl set /test/%m ${COREOS_PRIVATE_IPV4} (code=exited, status=0/SUCCESS)
 Main PID: 6802 (docker)
   CGroup: /system.slice/hello.service
           └─6802 /usr/bin/docker run --name test --rm busybox /bin/sh -c while true; do echo Hello World; sleep 1; done
```


### etcdctl

CoreOSに登録したUnitは、ExecStartPreでetcdにローカルの環境変数をセットしています。
`%m`のプレースホルダはMACHINEのIDになります。

``` bash ~/fleetctl_apps/hello/hello.service
ExecStartPre=/usr/bin/etcdctl set /test/%m ${COREOS_PRIVATE_IPV4}
```

`COREOS_PRIVATE_IPV4`は、[IDCFクラウドにCoreOSクラスタを構築する - Part5: 再セットアップ](/2014/07/19/idcf-coreos-cluster-re-setting-up/)で、CoreOSのインストール時に`/etc/environment`に記述しています。


etcdctlをクラスタ外部から使う場合、`--peers`オプションか`ETCDCTL_PEERS`環境変数に接続先のetcdのURLをしています。
lsコマンドで/testディレクトリを確認すると、`MACHINE`のIDのキーが作成されています。

``` bash
$ etcdctl --peers=http://10.1.0.174:4001 ls /test
/test/3b9bf346d5c64558bec638bb36aca48d
```

getコマンドで値を取得すると、ExecStartPreでsetしたIPアドレスを確認できます。

``` bash
$ etcdctl --peers=http://10.1.0.174:4001 get /test/3b9bf346d5c64558bec638bb36aca48d
10.1.1.90
```

### hello.serviceを削除

destroyコマンドを使い、登録したUnitを削除します。

``` bash
$ fleetctl destroy hello.service
Destroyed Job hello.service
```

list-unitsコマンドを使い、Unitが削除されたことを確認します。

``` bash
$ fleetctl list-units
UNIT    DSTATE  TMACHINE        STATE   MACHINE ACTIVE
```

