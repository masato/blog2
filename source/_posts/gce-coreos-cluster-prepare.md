title: 'GCEにCoreOSクラスタを構築する'
date: 2014-05-12 23:39:35
tags:
 - GCE
 - CoreOS
 - cloud-config
description: CoreOSでcloud-configの使い方を勉強しています。coreos-vagrantにあるサンプルを動かしたいのですが、手元にVagrant+VirtualBoxの環境がないため、GCE上に構築しました。
---

* `Update 2014-06-28`: [IDCFクラウドにCoreOSクラスタを構築する - Part1: CoreOSの準備](/2014/06/28/idcf-coreos-cluster)
* `Update 2014-07-10`: [IDCFクラウドにCoreOSクラスタを構築する - Part2: CoreOSのディスカバリ](/2014/07/10/idcf-coreos-cluster-discovery)
* `Update 2014-07-15`: [IDCFクラウドにCoreOSクラスタを構築する - Part3: クラスタをセットアップ](/2014/07/15/idcf-coreos-cluster-setting-up)


CoreOSでcloud-configの使い方を勉強しています。[coreos-vagrant](https://github.com/coreos/coreos-vagrant)にあるサンプルを動かしたいのですが、手元にVagrant+VirtualBoxの環境がないため、GCE上に構築しました。


<!-- more -->

### Tokenの取得
discovery.etcd.ioからTokenを取得します。

``` bash
$ curl https://discovery.etcd.io/new
https://discovery.etcd.io/xxxxxxec30bcd32828da204f4a8c5e67
```

### cloud-config.yaml
最初にプロジェクトの作成をします。

``` bash
$ mkdir -p ~/gce_apps/fleet_apps
$ cd !$
```
discovery.etcd.io取得したTokenをdiscoveryに使用します。

``` yaml ~/gce_apps/fleet_apps/cloud-config.yaml
#cloud-config

coreos:
  etcd:
      discovery: https://discovery.etcd.io/xxxxxxec30bcd32828da204f4a8c5e67
      addr: $public_ipv4:4001
      peer-addr: $public_ipv4:7001
  units:
    - name: etcd.service
      command: start
    - name: fleet.service
      command: start
      runtime: no
      content: |
        [Unit]
        Description=fleet

        [Service]
        Environment=FLEET_PUBLIC_IP=$public_ipv4
        ExecStart=/usr/bin/fleet
```

### CoreOS 310.1.0イメージの追加

[Running CoreOS on Google Compute Engine](http://coreos.com/docs/running-coreos/cloud-providers/google-compute-engine/)に記載している、`beta channel`のイメージを、`Google Cloud Storage`から自分のアカウントに追加します。
``` bash
$ gcutil addimage --description="CoreOS 310.1.0" coreos-v310-1-0 gs://storage.core-os.net/coreos/amd64-usr/beta/coreos_production_gce.tar.gz
```

### CoreOSクラスタの作成
複数のインスタンスを同時に作成すると、discoveryにタイムアウトしてしまうので、1インスタンスずつ作成します。

``` bash
$ gcutil addinstance --image=coreos-v310-1-0 --persistent_boot_disk --zone=us-central1-a --machine_type=n1-standard-1 --metadata_from_file=user-data:cloud-config.yaml core1
INFO: Resolved coreos-v310-1-0 to coreos-v310-1-0
+-------+----------------+---------------+---------------+---------+
| name  | network-ip     | external-ip   | zone          | status  |
+-------+----------------+---------------+---------------+---------+
| core1 | xx.240.118.255 | xxx.59.81.232 | us-central1-a | RUNNING |
+-------+----------------+---------------+---------------+---------+
```

クラスタに追加されるまで、まってから、etcdを確認します。1台追加されました。

``` bash
$ curl https://discovery.etcd.io/xxxxxxec30bcd32828da204f4a8c5e67
{"action":"get","node":{"key":"/_etcd/registry/xxxxxxec30bcd32828da204f4a8c5e67","dir":true,"nodes":[{"key":"/_etcd/registry/xxxxxxec30bcd32828da204f4a8c5e67/179b2d980b774f42a58c7b9692215cbe","value":"http://xxx.59.81.232:7001","expiration":"2014-05-19T14:22:25.280367915Z","ttl":604752,"modifiedIndex":16671994,"createdIndex":16671994}],"modifiedIndex":16671508,"createdIndex":16671508}}
```

core1にSSH接続して、fleetctlコマンドでインスタンスを一覧します。

``` bash
$ gcutil  ssh --ssh_user=core core1
$ fleetctl list-machines -l
MACHINE                                 IP              METADATA
128b69cc-8736-47c5-b0dc-caa07cff8a5b    xxx.59.81.232   -
```

クラスタに追加されたのを確認してから2台目、3台目を作成します。

``` bash
$ gcutil addinstance --image=coreos-v310-1-0 --persistent_boot_disk --zone=us-central1-a --machine_type=n1-standard-1 --metadata_from_file=user-data:cloud-config.yaml core2
$ gcutil addinstance --image=coreos-v310-1-0 --persistent_boot_disk --zone=us-central1-a --machine_type=n1-standard-1 --metadata_from_file=user-data:cloud-config.yaml core3
```

2台目、3台目も、クラスタに追加されるまで、それぞれ30秒くらい待ちました。

``` bash
$ curl https://discovery.etcd.io/xxxxxxec30bcd32828da204f4a8c5e67
{"action":"get","node":{"key":"/_etcd/registry/xxxxxxec30bcd32828da204f4a8c5e67","dir":true,"nodes":[{"key":"/_etcd/registry/xxxxxxec30bcd32828da204f4a8c5e67/179b2d980b774f42a58c7b9692215cbe","value":"http://xxx.59.81.232:7001","expiration":"2014-05-19T14:22:25.280367915Z","ttl":604271,"modifiedIndex":16671994,"createdIndex":16671994},{"key":"/_etcd/registry/xxxxxxec30bcd32828da204f4a8c5e67/43e2ad871a5c4bab95acfecda888627e","value":"http://xx.236.53.190:7001","expiration":"2014-05-19T14:28:22.683998037Z","ttl":604628,"modifiedIndex":16673607,"createdIndex":16673607},{"key":"/_etcd/registry/xxxxxxec30bcd32828da204f4a8c5e67/5f8680b47fa548d09a7403fa7f47567a","value":"http://xxx.223.233.121:7001","expiration":"2014-05-19T14:31:03.329011319Z","ttl":604789,"modifiedIndex":16674330,"createdIndex":16674330}],"modifiedIndex":16671508,"createdIndex":16671508}}
```

core1にSSH接続して、fleetctlコマンドでインスタンスを一覧します。

``` bash
$ gcutil  ssh --ssh_user=core core3
$ fleetctl list-machines -l
MACHINE                                 IP              METADATA
128b69cc-8736-47c5-b0dc-caa07cff8a5b    xxx.59.81.232   -
d92e436b-8dc8-441c-8842-fa5b7b4a2840    xx.236.53.190   -
dfe3051c-b2c1-4ce6-aeb0-aef016d4423b    xxx.223.233.121 -
```

### CoreOSクラスタ外からfleetctlで接続する
作業用のUbuntuにfleet をセットアップします。

``` bash
$ git clone https://github.com/coreos/fleet.git
$ cd fleet
$ ./build
Building fleet...
Building fleetctl...
```
endpointに、core1のexternal-ipを指定します。

``` bash
$ bin/fleetctl --endpoint 'http://xxx.59.81.232:4001'  list-machines -l
MACHINE IP              METADATA
        xxx.59.81.232   -
        xx.236.53.190   -
        xxx.223.233.121 -
```

### まとめ
gcutilコマンドが便利なので、coreos-vagrantと同じくらい簡単にクラスタができました。
GCEとCoreOSなので、インスタンスの起動も速いです。
CoreOSやMesosを触っていると、Datacenter-as-a-Computerというのが、今後のクラウドの使い方のベースになる気がして、
OSやコンピューターの概念の規模感がだいぶ変わってきました。

