title: "Kubernetes in CoreOS with Rudder on IDCFクラウド - Part4: GuestBook example"
date: 2014-09-05 09:15:40
tags:
 - Docker管理
 - Kubernetes
 - CoreOS
 - fleet
 - Rudder
 - IDCFクラウド
 - Go
description: CoreOSクラスタにインストールしたKubernetesに、GuestBook exampleとデプロイして、KubernetesとCoreOSとDockerの動きを確認していきます。Docker管理システムの中でもアーキテクチャがわかりやすく、動作も安定しています。
---

* `Update 2014-10-02`: [Kubernetes on CoreOS with Fleet and flannel on IDCFクラウド - Part7: flannel版GuestBook](/2014/10/02/kubernetes-fleet-flannel-guestbook/)
* `Update 2014-09-27`: [Kubernetes on CoreOS with Fleet and flannel on IDCFクラウド - Part6: Rudder renamed to flannel](/2014/09/27/kubernetes-fleet-rudder-renamed-to-flannel/)
* `Update 2014-09-22`: [Kubernetes on CoreOS with Fleet and Rudder on IDCFクラウド - Part5: FleetとRudderで再作成する](/2014/09/22/kubernetes-fleet-rudder-on-idcf/)

[CoreOSクラスタにインストールしたKubernetes](/2014/09/04/kubernetes-in-coreos-on-idcf-with-rudder/)に、[GuestBook example](https://github.com/GoogleCloudPlatform/kubernetes/blob/master/examples/guestbook/README.md)とデプロイして、KubernetesとCoreOSとDockerの動きを確認していきます。Docker管理システムの中でもアーキテクチャがわかりやすく、動作も安定しています。

<!-- more -->


### 作業マシンにfleetインストール

Kubernetesクラスタの外側の作業マシンの準備をします。
作業マシンからはCoreOSホストに普通にSSH接続できる状態です。

``` bash
$ eval `ssh-agent`
$ ssh-add ~/.ssh/deis
$ ssh core@10.1.0.44
```

fleetのビルドにはGo1.2以上が必要なのでGoをインストールします。

``` bash
$ sudo apt-get update
$ sudo apt-get install golang
$ go env GOROOT
/usr/lib/go
$ echo "export GOPATH=$HOME/gocode" >> ~/.profile
$ echo "export PATH=$PATH:$GOPATH/bin"  >> ~/.profile
$ source ~/.profile
$ go version
go version go1.2.1 linux/amd64
```

fleetをインストールします。

``` bash
$ git clone https://github.com/coreos/fleet.git
$ cd fleet
$ ./build
$ sudo mv bin/fleetctl /usr/local/bin/
```

トンネル用の環境変数を設定して、fleetctlコマンドでCoreOSホストをリストします。

``` bash
$ export FLEETCTL_TUNNEL=10.1.0.44
$ fleetctl list-machines
MACHINE		IP		METADATA
4b3c0589...	10.1.0.44	-
f696f27e...	10.1.3.13	-
fdb23eca...	10.1.1.140	-
```

fleetctl経由で、SSH接続の確認をします。

``` bash
$ fleetctl ssh 4b3c0589
CoreOS (stable)
```

### 作業マシンにkubecfgをインストール

作業マシンからCoreOSにインストールしたapiserverへのSSHトンネルを作成します。

``` bash
$ eval `ssh-agent`
$ ssh-add ~/.ssh/deis
$ ssh -f -nNT -L 8080:127.0.0.1:8080 core@10.1.1.140
```

作業マシンからKubernetesへの接続を確認します。

```
$ curl -s http://localhost:8080
<html><body>Welcome to Kubernetes</body></html>
```

作業マシンへ、kubecfg クライアントをインストールします。

``` bash
$ sudo wget https://storage.googleapis.com/kubernetes/kubecfg -O /usr/local/bin/kubecfg
$ sudo chmod +x /usr/local/bin/kubecfg
```

### GuestBook exampleをgit clone

GuestBook exampleを`git clone`します。

``` bash
$ git clone https://github.com/GoogleCloudPlatform/kubernetes.git
$ cd ~/kubernetes/examples/guestbook/
```

### Redis Master Podの作成

`Redis Master Pod`の定義ファイルです。

``` json ~/kubernetes/examples/guestbook/redis-master.json
{
  "id": "redis-master-2",
  "kind": "Pod",
  "apiVersion": "v1beta1",
  "desiredState": {
    "manifest": {
      "version": "v1beta1",
      "id": "redis-master-2",
      "containers": [{
        "name": "master",
        "image": "dockerfile/redis",
        "ports": [{
          "containerPort": 6379,
          "hostPort": 6379
        }]
      }]
    }
  },
  "labels": {
    "name": "redis-master"
  }
}
```

Podの作成をします。

``` bash
$ kubecfg -c redis-master.json create pods
I0903 13:02:30.706396 24249 request.go:287] Waiting for completion of /operations/1
Name                Image(s)            Host                Labels
----------          ----------          ----------          ----------
redis-master-2      dockerfile/redis    10.1.0.44/          name=redis-master
```

CoreOSホストにSSH接続して、psを確認します。

``` bash
$ fleetctl ssh 4b3c0589
$ docker ps
CONTAINER ID        IMAGE                     COMMAND             CREATED              STATUS              PORTS                    NAMES
dc14f8169701        kubernetes/pause:latest   /pause              About a minute ago   Up About a minute   0.0.0.0:6379->6379/tcp   k8s--net.51c76ba5--redis_-_master_-_2.etcd--a28daf81
```

### Redis Master Serviceの作成

KubernetesのServiceは、Podへの通信をプロキシーするロードバランサーに該当します。

Serviceの定義ファイルです。

``` json ~/kubernetes/examples/guestbook/redis-master-service.json
{
  "id": "redismaster",
  "kind": "Service",
  "apiVersion": "v1beta1",
  "port": 10000,
  "selector": {
    "name": "redis-master"
  }
}
```

servicesの作成をします。

``` bash
$ kubecfg -c redis-master-service.json create services
Name                Labels              Selector            Port
----------          ----------          ----------          ----------
redismaster                             name=redis-master   10000
```

### Redis Slave ReplicationControllerの作成

ReplicationControllerはレプリカされたPodを作成します。

ReplicationControllerの定義ファイルです。レプリカ数は2になっています。

``` json ~/kubernetes/examples/guestbook/redis-slave-controller.json
{
  "id": "redisSlaveController",
  "kind": "ReplicationController",
  "apiVersion": "v1beta1",
  "desiredState": {
    "replicas": 2,
    "replicaSelector": {"name": "redisslave"},
    "podTemplate": {
      "desiredState": {
         "manifest": {
           "version": "v1beta1",
           "id": "redisSlaveController",
           "containers": [{
             "name": "slave",
             "image": "brendanburns/redis-slave",
             "ports": [{"containerPort": 6379, "hostPort": 6380}]
           }]
         }
       },
       "labels": {"name": "redisslave"}
      }},
  "labels": {"name": "redisslave"}
}
```

replicationControllersを作成します。

``` bash
$ kubecfg -c redis-slave-controller.json create replicationControllers
I0903 13:09:07.445538 30248 request.go:287] Waiting for completion of /operations/3
Name                   Image(s)                   Selector            Replicas
----------             ----------                 ----------          ----------
redisSlaveController   brendanburns/redis-slave   name=redisslave     2
```

ここまで作成したPodをリストします。redis-master x1 とredis-slave x2のPodが作成されました。

``` bash
$ kubecfg list /pods
Name                                   Image(s)                   Host                Labels
----------                             ----------                 ----------          ----------
redis-master-2                         dockerfile/redis           10.1.0.44/          name=redis-master
0abd9a1f-3320-11e4-a6f3-02001f3a0128   brendanburns/redis-slave   10.1.0.44/          name=redisslave,replicationController=redisSlaveController
0abdead2-3320-11e4-a6f3-02001f3a0128   brendanburns/redis-slave   10.1.3.13/          name=redisslave,replicationController=redisSlaveController
```

### Redis Slave Serviceの作成

`Redis Master`と同様に、`Redis Slave Pod`のロードバランサーを作成します。

Serviceの定義ファイルです。

``` json ~/kubernetes/examples/guestbook/redis-slave-service.json
{
  "id": "redisslave",
  "kind": "Service",
  "apiVersion": "v1beta1",
  "port": 10001,
  "labels": {
    "name": "redisslave"
  },
  "selector": {
    "name": "redisslave"
  }
}
```

servicesを作成します。

``` bash
$ kubecfg -c redis-slave-service.json create services
Name                Labels              Selector            Port
----------          ----------          ----------          ----------
redisslave          name=redisslave     name=redisslave     10001
```

### Frontend Podの作成

フロントエンドのPHPアプリ用の、レプリカされたPodを作成します。

ReplicationControllerの定義ファイルです。レプリカ数は3です。

``` json ~/kubernetes/examples/guestbook/frontend-controller.json
{
  "id": "frontendController",
  "kind": "ReplicationController",
  "apiVersion": "v1beta1",
  "desiredState": {
    "replicas": 3,
    "replicaSelector": {"name": "frontend"},
    "podTemplate": {
      "desiredState": {
         "manifest": {
           "version": "v1beta1",
           "id": "frontendController",
           "containers": [{
             "name": "php-redis",
             "image": "brendanburns/php-redis",
             "ports": [{"containerPort": 80, "hostPort": 8000}]
           }]
         }
       },
       "labels": {"name": "frontend"}
      }},
  "labels": {"name": "frontend"}
}
```

replicationControllersを作成します。

``` bash
$ kubecfg -c frontend-controller.json create replicationControllers
I0903 13:13:46.143184 01871 request.go:287] Waiting for completion of /operations/7
Name                 Image(s)                 Selector            Replicas
----------           ----------               ----------          ----------
frontendController   brendanburns/php-redis   name=frontend       3
```

### Podの確認

最終的に作成されたPodを一覧します。
redis-master x1, redis-slave x2, php-frontend x3が作成されました。

``` bash
$ kubecfg list pods
Name                                   Image(s)                   Host                Labels
----------                             ----------                 ----------          ----------
redis-master-2                         dockerfile/redis           10.1.0.44/          name=redis-master
0abd9a1f-3320-11e4-a6f3-02001f3a0128   brendanburns/redis-slave   10.1.0.44/          name=redisslave,replicationController=redisSlaveController
0abdead2-3320-11e4-a6f3-02001f3a0128   brendanburns/redis-slave   10.1.3.13/          name=redisslave,replicationController=redisSlaveController
b0db656d-3320-11e4-a6f3-02001f3a0128   brendanburns/php-redis     10.1.0.44/          name=frontend,replicationController=frontendController
b0db9a9e-3320-11e4-a6f3-02001f3a0128   brendanburns/php-redis     10.1.3.13/          name=frontend,replicationController=frontendController
b0dbb1cd-3320-11e4-a6f3-02001f3a0128   brendanburns/php-redis     10.1.1.140/         name=frontend,replicationController=frontendController
```

fleetからはCoreOSクラスタ構成のリストできますが、unitとしては認識されません。

``` bash
$ fleetctl list-machines
MACHINE         IP              METADATA
4b3c0589...     10.1.0.44       -
f696f27e...     10.1.3.13       -
fdb23eca...     10.1.1.140      -
$ fleetctl list-units
UNIT    MACHINE ACTIVE  SUB
```

### ブラウザから確認

`kubecfg list pods`でphp-frontendがどのCoreOSホストで起動しているか調べます。
3台のCoreOSホストのIPアドレスを指定して、ブラウザから確認します。

* http://10.1.0.44:8000
* http://10.1.3.13:8000
* http://10.1.1.140:8000
 
{% img center /2014/09/05/kubernetes-in-coreos-on-idcf-guestbook-example/guestbook.png %}

### まとめ

KubernetesではPodを分類する仕組みとしてLabelが重要な役割を果たしますが、まだよく理解できていません。
LabelでServiceとPodを紐付けるようですが、GuestBookサンプルでは同じ値のラベルが複数登場します。

Kubernetesのサンプルプリが動くようになったので、次回からアーキテクチャについて調べていこうと思います。
