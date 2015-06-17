title: 'Couchbase on Docker - Part3: Couchbase Sync Gatewayとクラスタ再構成'
date: 2015-01-09 21:34:28
tags:
 - CoreOS
 - Couchbase
 - CouchbaseServer
 - CouchbaseSyncGateway
 - CouchbaseMobile
 - IDCFクラウド
 - Xamarin
 - ionic
description: Couchbase Sync Gatewayを追加したクラスタ再設定して、Couchbase Mobileのサーバーサイドを構築します。ようやくサーバーサイドができあがりました。この後はXamarinやionicなどフロントエンド開発をしてCouchbase Mobileと連携したアプリを開発をします。
---

Couchbase Sync Gatewayを追加したクラスタ再設定して、Couchbase Mobileのサーバーサイドを構築します。ようやくサーバーサイドができあがりました。この後はXamarinやionicなどフロントエンド開発をしてCouchbase Mobileと連携したアプリを開発をします。

<!-- more -->

### fleetクラスタの再構築

前回[tleyden5iwx/couchbase-server-3.0.1](https://registry.hub.docker.com/u/tleyden5iwx/couchbase-server-3.0.1/)を使い構築したCouchbase Serverのfleetクラスタを、[tleyden5iwx/couchbase-sync-gateway](https://registry.hub.docker.com/u/tleyden5iwx/couchbase-sync-gateway/)を使って作り直します。

### fleet unitsの破棄

再構築前のunitsは以下のようになっています。

``` bash
$ fleetctl list-units
UNIT                                            MACHINE                 ACTIVE  SUB
couchbase_bootstrap_node.service                33bf6324.../10.3.0.7    active  running
couchbase_bootstrap_node_announce.service       33bf6324.../10.3.0.7    active  running
couchbase_node.1.service                        461b77ab.../10.3.0.193  active  running
couchbase_node.2.service                        80e8c0f2.../10.3.0.210  active  running
```

デプロイ済みのunitsをすべて破棄します。

``` bash
$ fleetctl destroy couchbase_bootstrap_node.service couchbase_bootstrap_node_announce.service couchbase_node.1.service couchbase_node.2.service 
Destroyed couchbase_bootstrap_node.service
Destroyed couchbase_bootstrap_node_announce.service
Destroyed couchbase_node.1.service
Destroyed couchbase_node.2.servic
$ fleetctl list-units
UNIT    MACHINE ACTIVE  SUB
```

### cloud-initのやり直し

すべての仮想マシンで`/var/lib/coreos-install/user_data/user_data`を編集してcloud-initをやり直します。CBFSディレクトリを作成する`write_files`を追記します。

``` yml /var/lib/coreos-install/user_data/user_data

- path: /var/lib/cbfs/data/.README
    owner: core:core
    permissions: 0644
    content: |
      CBFS files are stored here
```

新しいcloud-config.ymlは以下の通りです。

``` yml cloud-config.yml
#cloud-config
write_files:
  - path: /etc/environment
    content: |
      COREOS_PUBLIC_IPV4=
      COREOS_PRIVATE_IPV4=10.3.0.7
  - path: /etc/systemd/system/docker.service.d/increase-ulimit.conf
    owner: core:core
    permissions: 0644
    content: |
      [Service]
      LimitMEMLOCK=infinity
  - path: /var/lib/couchbase/data/.README
    owner: core:core
    permissions: 0644
    content: |
      Couchbase Data files are stored here
  - path: /var/lib/couchbase/index/.README
    owner: core:core
    permissions: 0644
    content: |
      Couchbase Index files are stored here
  - path: /var/lib/cbfs/data/.README
    owner: core:core
    permissions: 0644
    content: |
      CBFS files are stored here
coreos:
  etcd:
    discovery: https://discovery.etcd.io/5c8b6b2d75b706d24e37c2e202984a2f
    addr: 10.3.0.7:4001
    peer-addr: 10.3.0.7:7001
  units:
    - name: etcd.service
      command: start
    - name: fleet.service
      command: start
    - name: docker.service
      command: restart
    - name: timezone.service
      command: start
      content: |
        [Unit]
        Description=timezone
        [Service]
        Type=oneshot
        RemainAfterExit=yes
        ExecStart=/usr/bin/ln -sf ../usr/share/zoneinfo/Japan /etc/localtime
ssh_authorized_keys:
  - ssh-rsa AAAAB3Nza...
```

coreos-cloudinitを実行して上記のディレクトを作成します。

``` bash
$ sudo coreos-cloudinit --from-file /var/lib/coreos-install/user_data
```


アーキテクチャ図を[tleyden/sync-gateway-coreos](https://github.com/tleyden/sync-gateway-coreos)から転載します。

{% img center /2015/01/09/idcf-coreos-couchbase-sync-gateway-install/coreos-couchbase-sync-gateway.png %}

### sync-gw-cluster-init.shスクリプト


インストールスクリプトをダウンロードして実行権限をつけます。

[sync-gw-cluster-init.sh](https://github.com/tleyden/sync-gateway-coreos/blob/master/scripts/sync-gw-cluster-init.sh)

``` bash sync-gw-cluster-init.sh
# Args:
#   -n number of Sync Gateway nodes to start
#   -c the commit or branch to use.  If "image" is given, will use master branch at time of docker image build.
#   -b the name of the couchbase bucket to create, or leave this off if you don't need to create a bucket.  (optional)
#   -z the size of the bucket in megabytes, or leave this off if you don't need to create a bucket.  (optional)
#   -g the Sync Gateway config file or URL to use.
#   -v Couchbase Server version (3.0.1 or 2.2, or 0 to skip the couchbase server initialization)
#   -m number of couchbase nodes to start
#   -u the username and password as a single string, delimited by a colon (:)

usage="./sync-gw-cluster-init.sh -n 1 -c \"master\" -b \"todolite\" -z \"512\" -g \"http://foo.com/config.json\" -v 3.0.1 -m 3 -u \"user:passw0rd\""
```

sync-gw-cluster-init.shをダウンロードして実行権限をつけます。


``` bash
$ wget https://raw.githubusercontent.com/tleyden/sync-gateway-coreos/master/scripts/sync-gw-cluster-init.sh
$ chmod +x sync-gw-cluster-init.sh
```

環境変数をセットして、インストールスクリプトを実行します。

```
$ SG_CONFIG_URL=https://raw.githubusercontent.com/couchbaselabs/ToDoLite-iOS/master/sync-gateway-config.json
$ ./sync-gw-cluster-init.sh -n 1 -c master -b "todos" -z 512 -g $SG_CONFIG_URL -v 3.0.1 -m 3 -u user:passw0rd
Kick off couchbase cluster
--2015-01-09 12:26:26--  https://raw.githubusercontent.com/couchbaselabs/couchbase-server-docker/master/scripts/cluster-init.sh
Resolving raw.githubusercontent.com... 103.245.222.133
Connecting to raw.githubusercontent.com|103.245.222.133|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 1940 (1.9K) [text/plain]
Saving to: 'cluster-init.sh'

cluster-init.sh          100%[===================================>]   1.89K  --.-KB/s   in 0s

2015-01-09 12:26:27 (82.4 MB/s) - 'cluster-init.sh' saved [1940/1940]

Cloning into 'couchbase-server-docker'...
remote: Counting objects: 350, done.
remote: Compressing objects: 100% (132/132), done.
remote: Total 350 (delta 195), reused 350 (delta 195)
Receiving objects: 100% (350/350), 39.46 KiB | 0 bytes/s, done.
Resolving deltas: 100% (195/195), done.
Checking connectivity... done.
user:passw0rd
Kicking off couchbase server bootstrap node
Unit couchbase_bootstrap_node.service launched
Unit couchbase_bootstrap_node_announce.service launched on 33bf6324.../10.3.0.7
Kicking off  additional couchbase server nodes
Unit couchbase_node.2.service launched on 80e8c0f2.../10.3.0.210
Unit couchbase_node.1.service launched on 461b77ab.../10.3.0.193
Wait until Couchbase bootstrap node is up
Retrying...
Couchbase Server bootstrap ip: 10.3.0.7
Wait until 3 Couchbase Servers running
Retrying... 0 != 3
Retrying... 1 != 3
Retrying... 1 != 3
Retrying... 2 != 3
Done waiting: 3 Couchbase Servers are running
INFO: rebalancing . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . .
SUCCESS: rebalanced cluster
Create a bucket: todos with size: 512
SUCCESS: bucket-create
Done: created a bucket
https://raw.githubusercontent.com/couchbaselabs/ToDoLite-iOS/master/sync-gateway-config.json
master
Cloning into 'sync-gateway-coreos'...
remote: Counting objects: 101, done.
remote: Total 101 (delta 0), reused 0 (delta 0)
Receiving objects: 100% (101/101), 16.93 KiB | 0 bytes/s, done.
Resolving deltas: 100% (49/49), done.
Checking connectivity... done.
Unit sync_gw_node@1.service launched on f0f5fcc4.../10.3.0.47
Unit sync_gw_announce@1.service launched on f0f5fcc4.../10.3.0.47
Your couchbase server + sync gateway cluster is now active!
```

何らかの事象でインストールに失敗した場合は、いちどカレントのディレクトリをきれいにします。

``` bash
$ rm cluster-init.sh*
$ rm -fr couchbase-server-docker sync-gateway-coreos
$ SG_CONFIG_URL=https://raw.githubusercontent.com/couchbaselabs/ToDoLite-iOS/master/sync-gateway-config.json
$ ./sync-gw-cluster-init.sh -n 1 -c master -b "todos" -z 512 -g $SG_CONFIG_URL -v 3.0.1 -m 3 -u user:passw0rd
Kick off couchbase cluster
```

### fleet unitsの確認

fleet unitsをリストします。

``` bash
$ fleetctl list-units
UNIT                                            MACHINE                 ACTIVE  SUB
couchbase_bootstrap_node.service                33bf6324.../10.3.0.7    active  running
couchbase_bootstrap_node_announce.service       33bf6324.../10.3.0.7    active  running
couchbase_node.1.service                        461b77ab.../10.3.0.193  active  running
couchbase_node.2.service                        80e8c0f2.../10.3.0.210  active  running
sync_gw_announce@1.service                      f0f5fcc4.../10.3.0.47   active  running
sync_gw_node@1.service                          f0f5fcc4.../10.3.0.47   active  running
```

couchbase_bootstrap_nodeを含めてCouchbase Server が3台、Couchbase Sync Gatewayが1台作成できました。


Couchbase Sync Gatewayの起動を確認します。

``` bash
$ curl 10.3.0.47:4984
{"couchdb":"Welcome","vendor":{"name":"Couchbase Sync Gateway","version":1},"version":"Couchbase Sync Gateway/master(6bdb3f6)"}
```

パブリックIPアドレスから使う場合は、nodeの仮想マシンへのport forwardの設定が必要です。

プライベートIPアドレス
http://10.3.0.210:8091

パブリックIPアドレス
http://210.140.173.63:8091


* user: user
* passwd: passw0rd

CouchbaseSyncGateway Adminインタフェースはローカルのみアクセス可能です。sync_gw_nodeにログインしてから実行します。

```
$ fleetctl ssh f0f5fcc4
$ curl localhost:4985
{"ADMIN":true,"couchdb":"Welcome","vendor":{"name":"Couchbase Sync Gateway","version":1},"version":"Couchbase Sync Gateway/master(6bdb3f6)"}
```

以下のようにAdminポートの4985は、外部から接続できません

``` bash
$ curl 10.3.0.47:4985
curl: (7) Failed connect to 10.3.0.47:4985; Connection refused
```

確認

``` bash
$ curl localhost:4985/todos/
{"committed_update_seq":1,"compact_running":false,"db_name":"todos","disk_format_version":0,"instance_start_time":1420774094012079,"purge_seq":0,"update_seq":1}
```