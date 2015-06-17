title: "Kubernetes in CoreOS with Rudder on IDCFクラウド - Part1: マニュアルインストール"
date: 2014-08-20 01:12:33
tags:
 - Docker管理
 - Kubernetes
 - CoreOS
 - IDCFクラウド
 - SDN
description: Pamanaxに続いてDockerコンテナ管理ツールのKubernetesをIDCFクラウド上のCoreOSにインストールしてみます。今回は最初なので直接CoreOSにSSH接続してマニュアルでインストールします。CoreOSの世界ではcloud-configで初期設定をしたあとに直接変更を加えるのはお行儀が良くないので、インストールの確認ができたら、cloud-configでKubernetesをCoreOSクラスタにインストールしたいと思います。
---

* `Update 2014-10-02`: [Kubernetes on CoreOS with Fleet and flannel on IDCFクラウド - Part7: flannel版GuestBook](/2014/10/02/kubernetes-fleet-flannel-guestbook/)
* `Update 2014-09-27`: [Kubernetes on CoreOS with Fleet and flannel on IDCFクラウド - Part6: Rudder renamed to flannel](/2014/09/27/kubernetes-fleet-rudder-renamed-to-flannel/)
* `Update 2014-09-22`: [Kubernetes on CoreOS with Fleet and Rudder on IDCFクラウド - Part5: FleetとRudderで再作成する](/2014/09/22/kubernetes-fleet-rudder-on-idcf/)
* `Update 2014-09-05`: [Kubernetes in CoreOS with Rudder on IDCFクラウド - Part4: GuestBook example](/2014/09/05/kubernetes-in-coreos-on-idcf-guestbook-example/)
* `Update 2014-09-04`: [Kubernetes in CoreOS with Rudder on IDCFクラウド - Part3: RudderでSDN風なこと](/2014/09/04/kubernetes-in-coreos-on-idcf-with-rudder/)
* `Update 2014-08-31`: [Kubernetes in CoreOS with Rudder on IDCFクラウド - Part2: クラスタ再考](/2014/08/31/kubernetes-on-coreos-cluster/)

[Pamanax](/2014/08/19/panamax-in-coreos-on-idcf/)に続いてDockerコンテナ管理ツールのKubernetesをIDCFクラウド上のCoreOSにインストールしてみます。今回は最初なので直接CoreOSにSSH接続してマニュアルでインストールします。

CoreOSの世界ではcloud-configで初期設定をしたあとに直接変更を加えるのはお行儀が良くないので、インストールの確認ができたら、cloud-configでKubernetesをCoreOSクラスタにインストールしたいと思います。

<!-- more -->

### CoreOSのインストール

CoreOSを[ISOからインストール](/2014/08/18/idcf-coreos-install-from-iso/)して、コンソールからパスワード変更をします。

``` bash
$ sudo passwd core
```

パスワードでSSH接続できるようになりました。CoreOSインスタンスにSSH接続します。

``` bash
$ ssh core@10.1.1.120
```

CoreOSインスタンスにアサインされたipアドレスを確認します。

``` bash
$ ip a
...
2: ens32: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP qlen 1000
    link/ether 02:00:30:55:01:12 brd ff:ff:ff:ff:ff:ff
    inet 10.1.1.120/22 brd 10.1.3.255 scope global ens32
       valid_lft forever preferred_lft forever
    inet6 fe80::30ff:fe55:112/64 scope link
       valid_lft forever preferred_lft forever
```

cloud-config.ymlにIPアドレスを指定して、あとは最低限の設定のみ行います。
`stop-update-engine.service`のunitも明示的に書かないとKubernetesのインストール時におこられます。

``` yaml ~/coreos/cloud-config.yml
 #cloud-config

ssh_authorized_keys:
  - ssh-rsa AAAAB3NzaC1... deis

coreos:
  etcd:
    discovery: https://discovery.etcd.io/8cb9184d20c460c4be658a846b6cab82
    addr: 10.1.1.120:4001
    peer-addr: 10.1.1.120:7001
  units:
    - name: etcd.service
      command: start
    - name: fleet.service
      command: start
    - name: stop-update-engine.service
      command: start
      content: |
        [Unit]
        Description=stop update-engine

        [Service]
        Type=oneshot
        ExecStart=/usr/bin/systemctl stop update-engine.service
        ExecStartPost=/usr/bin/systemctl mask update-engine.service

write_files:
  - path: /etc/environment
    permissions: 0644
    content: |
      COREOS_PUBLIC_IPV4=
      COREOS_PRIVATE_IPV4=10.1.1.120
  - path: /etc/profile.d/nse-function.sh
    permissions: 0755
    content: |
      function nse() {
        sudo nsenter --pid --uts --mount --ipc --net --target $(docker inspect --format="&#123;&#123; .State.Pid &#125;&#125;" $1)
      }
```

/dev/sdaにCoreOSをインストールします。

``` bash
$ sudo coreos-install -d /dev/sda -C stable -c ./cloud-config.yml
...
Installing cloud-config...
Success! CoreOS stable 367.1.0 is installed on /dev/sda
```

rebootします。

``` bash
$ sudo reboot
```

公開鍵でSSH接続できることを確認します。

``` bash
$ eval `ssh-agent`
$ ssh-add ~/.ssh/deis
$ ssh -A core@10.1.1.120
```


### Kubernetesのインストール

[kubernetes-coreos](https://github.com/kelseyhightower/kubernetes-coreos)を読みながらインストールをしていきます。まずはCoreOSにログインして、Kubernetesバイナリのインストールします。

``` bash
$ sudo mkdir -p /opt/bin
$ sudo wget https://storage.googleapis.com/kubernetes/binaries.tar.gz
$ sudo tar -xvf binaries.tar.gz -C /opt/bin
apiserver
controller-manager
kubecfg
kubelet
proxy
```


Kubernetes のunitsを`git clone`してコピーします。

``` bash
$ git clone https://github.com/kelseyhightower/kubernetes-coreos.git
$ ls kubernetes-coreos/units/
apiserver.service           docker.service               kubelet.service
controller-manager.service  download-kubernetes.service  proxy.service
$ sudo cp kubernetes-coreos/units/* /etc/systemd/system/
```


Kubernetes サービスを開始します。

``` bash
$ sudo systemctl start apiserver
$ sudo systemctl start controller-manager
$ sudo systemctl start kubelet
$ sudo systemctl start proxy
```


### 作業マシンにkubecfg クライアントのインストール

リモートの作業マシンで作業するbyobuでシェルマルチプレクサを開いておきます。

``` bash
$ byobu
```

作業マシンからCoreOSにインストールしたapiserverへのSSHトンネルを作成します。

``` bash
$ eval `ssh-agent`
$ ssh-add ~/.ssh/deis
$ ssh -f -nNT -L 8080:127.0.0.1:8080 core@10.1.1.120
```

作業マシンからKubernetesへの接続を確認します。

``` bash
$ curl -s http://localhost:8080
<html><body>Welcome to Kubernetes</body></html>
```

作業マシンへ、kubecfg クライアントをインストールします。

``` bash
$ sudo wget https://storage.googleapis.com/kubernetes/kubecfg -O /usr/local/bin/kubecfg
$ sudo chmod +x /usr/local/bin/kubecfg
```

kubecfgコマンドの起動確認をします。

``` bash
$ /usr/local/bin/kubecfg list /pods
kubecfg list /pods
Name                Image(s)            Host                Labels
----------          ----------          ----------          ----------
```

### Redis Pod の作成

Podとは、コンテナをグループにした概念です。Podファイル用のディレクトリを作成します。

``` bash
$ mkdir ~/kubernetes-coreos/pods
$ cd !$
```

``` json ~/kubernetes-coreos/pods/redis.json
{
  "id": "redis",
  "desiredState": {
    "manifest": {
      "version": "v1beta1",
      "id": "redis",
      "containers": [{
        "name": "redis",
        "image": "dockerfile/redis",
        "ports": [{
          "containerPort": 6379,
          "hostPort": 6379 
        }]
      }]
    }
  },
  "labels": {
    "name": "redis"
  }
}
```

Redis Podの作成をします。2分ちょっと待ちます。

``` bash
$ /opt/bin/kubecfg -h http://127.0.0.1:8080 -c ~/kubernetes-coreos/pods/redis.json create /pods
...
I0819 03:56:12.709765 00662 request.go:287] Waiting for completion of /operations/1
Name                Image(s)            Host                Labels
----------          ----------          ----------          ----------
redis               dockerfile/redis    /                   name=redis
```

### Redis Pod の確認

`docker ps`でコンテナの起動を確認します。

``` bash
$ docker ps
CONTAINER ID        IMAGE                     COMMAND                CREATED             STATUS              PORTS                    NAMES
5d9727604f59        dockerfile/redis:latest   redis-server /etc/re   14 minutes ago      Up 14 minutes                                k8s--redis.d91677c6--redis.etcd--1575910b
8cd1158abdf9        kubernetes/pause:latest   /pause                 15 minutes ago      Up 15 minutes       0.0.0.0:6379->6379/tcp   k8s--net.51c76ba5--redis.etcd--c5dddb70
```

docker0インターフェース(172.17.42.1)へ、redis-cliのdisposableコンテナから接続します。

``` bash
$ docker run -t -i --rm  dockerfile/redis /usr/local/bin/redis-cli -h 172.17.42.1
172.17.42.1:6379>
```

リモートの作業マシンからもリストしてみます。

``` bash
$ /usr/local/bin/kubecfg list /pods
Name                Image(s)            Host                Labels
----------          ----------          ----------          ----------
redis               dockerfile/redis    127.0.0.1/          name=redis
```

### Redis Pod の削除

確認がおわったら、Podをdeleteします。

``` bash
$ /opt/bin/kubecfg -h http://127.0.0.1:8080 delete /pods/redis
Status
----------
success
```

listしてpodsにエントリがないことを確認します。

``` bash
$ /opt/bin/kubecfg -h http://127.0.0.1:8080 list /pods
Name                Image(s)            Host                Labels
----------          ----------          ----------          ----------
```


