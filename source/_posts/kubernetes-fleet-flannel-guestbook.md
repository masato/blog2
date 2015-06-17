title: "Kubernetes on CoreOS with Fleet and flannel on IDCFクラウド - Part7: flannel版GuestBook example"
date: 2014-10-02 01:17:11
tags:
 - Docker管理
 - Kubernetes
 - CoreOS
 - fleet
 - flannel
 - SDN
 - IDCFクラウド
description: Kubernetes on CoreOS with Fleet and flannelの構成でIDCFクラウドに再インストールしたk8sにGuestBookをデプロイします。手順はRudderのときと同じなので問題はないのですが、fleetクラスタとk8sクラスタの構成が一致せず、どちらかがemptyになる状態になりました。CoreOSを手動でrebootしたあと、flannelとdockerのunitsを個別でrestartして解決したのですが、systemdの依存関係を見直す必要があります。
---


[Kubernetes on CoreOS with Fleet and flannel](/2014/09/27/kubernetes-fleet-rudder-renamed-to-flannel/)の構成で再インストールしたk8sに[GuestBook](https://github.com/GoogleCloudPlatform/kubernetes/tree/master/examples/guestbook)をデプロイします。

手順は[Rudder](/2014/09/05/kubernetes-in-coreos-on-idcf-guestbook-example/)のときと同じなので問題はないのですが、fleetクラスタとk8sクラスタの構成が一致せず、どちらかがemptyになる状態になりました。CoreOSを手動でrebootしたあと、flannelとdockerのunitsを個別でrestartして解決したのですが、systemdの起動と依存関係の順番を見直す必要があります。

<!-- more -->


### kubecfgでlistの確認

kubernetesはすでにclone済みなのでpullします。

``` bash
$ cd ~/kubernetes
$ git pull
```

[前回](/2014/09/27/kubernetes-fleet-rudder-renamed-to-flannel/)構築したk8sの3台構成minionsを作業マシンから確認します。

``` bash
$ export KUBERNETES_MASTER="http://10.1.1.99:8080"
$ kubecfg -version
Kubernetes v0.2-146-gc47dca5dbb9371
$ kubecfg list /minions
Minion identifier
----------
10.1.0.207
10.1.1.99
10.1.3.229
```

### fleetctlでlistできない

etcdサーバー用の環境変数設定を設定します。
 
``` bash
$ export ETCDCTL_PEERS=http://10.1.3.124:4001
$ export FLEETCTL_ENDPOINT=http://10.1.3.124:4001
```

kubecfgでは、3台のminionsがリストできましたが、fleetctlでetcdのmachinesしかリストできません。

``` bash
$ export ETCDCTL_PEERS=http://10.1.3.124:4001
$ export FLEETCTL_ENDPOINT=http://10.1.3.124:4001
$ fleetctl list-machines
MACHINE         IP              METADATA
1ee3b79b...     10.1.3.124      role=etcd
```

### CoreOSをrebootしたらkubecfgでlistできなくなった

とりあえずfleetクラスタのmachinesをrebootしてみます。

``` bash
$ eval `ssh-agent`
$ ssh-add ~/.ssh/deis
$ ssh core@10.1.0.207 sudo reboot
$ ssh core@10.1.1.99  sudo reboot
$ ssh core@10.1.3.229  sudo reboot
```

fleetctlでリストできるようになりました。

``` bash
$ fleetctl list-machines
MACHINE         IP              METADATA
1ee3b79b...     10.1.3.124      role=etcd
2f706a2d...     10.1.1.99       role=kubernetes
6d444a3b...     10.1.3.229      role=kubernetes
ad6fa03e...     10.1.0.207      role=kubernetes
```

今度はminionsのリストは失敗します。

``` bash
$ kubecfg list /minions
F1001 20:34:22.247104 23514 kubecfg.go:320] Got request error: Get http://10.1.1.99:8080/api/v1beta1/minions?labels=: dial tcp 10.1.1.99:8080: connection refused
```

### flannelとdockerのunitsをrestartする

`Kubernetes API Server`のmachineにSSH接続します。

``` bash
$ ssh core@10.1.1.99
CoreOS (alpha)
Update Strategy: No Reboots
Failed Units: 1
  docker.service
```

dockerの起動に失敗しています。journalctlでdockerのunitのログを確認します。

``` bash
$ journalctl -u docker -l
...
Oct 01 11:17:03 i-669-71792-VM systemd[1]: Starting Docker Application Container Engine...
Oct 01 11:17:03 i-669-71792-VM systemd[1]: Failed to load environment files: No such file or directory
Oct 01 11:17:03 i-669-71792-VM systemd[1]: docker.service failed to run 'start-pre' task: No such file or director
Oct 01 11:17:03 i-669-71792-VM systemd[1]: Failed to start Docker Application Container Engine.
```

docker.serviceファイルを確認します。

``` bash
$ systemctl cat docker.service
# /etc/systemd/system/docker.service
[Unit]
After=flannel.service
Wants=flannel.service
Description=Docker Application Container Engine
Documentation=http://docs.docker.io

[Service]
EnvironmentFile=/run/flannel/subnet.env
ExecStartPre=/bin/mount --make-rprivate /
ExecStart=/usr/bin/docker -d --bip=${FLANNEL_SUBNET} --mtu=${FLANNEL_MTU} -s=btrfs -H fd://

[Install]
WantedBy=multi-user.target
```

flannelが作成しているEnvironmentFileの/run/flannel/subnet.envファイルが見つからないようです。
flannelとdockerを順番にrestartすると、dockerが起動しました。

``` bash
$ sudo systemctl restart flannel
$ sudo systemctl restart docker
$ systemctl status docker
● docker.service - Docker Application Container Engine
   Loaded: loaded (/etc/systemd/system/docker.service; disabled)
   Active: active (running) since Wed 2014-10-01 11:44:00 UTC; 2min 25s ago
     Docs: http://docs.docker.io
  Process: 1017 ExecStartPre=/bin/mount --make-rprivate / (code=exited, status=0/SUCCESS)
 Main PID: 1019 (docker)
   CGroup: /system.slice/docker.service
           └─1019 /usr/bin/docker -d --bip=10.0.52.1/24 --mtu=1472 -s=btrfs -H fd://

Oct 01 11:44:00 i-669-71792-VM docker[1019]: [info] Listening for HTTP on fd ()
Oct 01 11:44:01 i-669-71792-VM docker[1019]: [1d106490.init_networkdriver()] getting iface addr
Oct 01 11:44:01 i-669-71792-VM docker[1019]: [1d106490] -job init_networkdriver() = OK (0)
Oct 01 11:44:01 i-669-71792-VM docker[1019]: [info] Loading containers:
Oct 01 11:44:01 i-669-71792-VM docker[1019]: [info] : done.
Oct 01 11:44:01 i-669-71792-VM docker[1019]: [1d106490] +job acceptconnections()
Oct 01 11:44:01 i-669-71792-VM docker[1019]: [1d106490] -job acceptconnections() = OK (0)
Oct 01 11:44:01 i-669-71792-VM docker[1019]: [info] GET /v1.14/containers/json
Oct 01 11:44:01 i-669-71792-VM docker[1019]: [1d106490] +job containers()
Oct 01 11:44:01 i-669-71792-VM docker[1019]: [1d106490] -job containers() = OK (0)
```

同様に残りの10.1.3.229と10.1.0.207もrestartします。
fleetctlとkubecfgの両方からリストができるようになりました。

``` bash
$ fleetctl list-machines
MACHINE         IP              METADATA
1ee3b79b...     10.1.3.124      role=etcd
2f706a2d...     10.1.1.99       role=kubernetes
6d444a3b...     10.1.3.229      role=kubernetes
ad6fa03e...     10.1.0.207      role=kubernetes
$ kubecfg list /minions
Minion identifier
----------
10.1.0.207
10.1.1.99
10.1.3.229
```

### GuestBookのデプロイ

SSHトンネルを作成します。

``` bash
$ eval `ssh-agent`
$ ssh-add ~/.ssh/deis
$ ssh -f -nNT -L 8080:127.0.0.1:8080 core@10.1.1.99
$ curl -s http://localhost:8080
<html><body>Welcome to Kubernetes</body></html>
```

git pullしたディレクトリに移動します。

``` bash
$ cd ~/kubernetes/examples/guestbook
```

順番にPodsを作成していきます。

``` bash
$ kubecfg -c redis-master.json create pods
ID                  Image(s)            Host                Labels              Status
----------          ----------          ----------          ----------          ----------
redis-master-2      dockerfile/redis    /                   name=redis-master   Waiting
$ kubecfg -c redis-master-service.json create services
ID                  Labels              Selector            Port
----------          ----------          ----------          ----------
redismaster                             name=redis-master   10000
$ kubecfg -c redis-slave-controller.json create replicationControllers
$ kubecfg -c redis-slave-service.json create services
$ kubecfg -c frontend-controller.json create replicationControllers
ID                   Image(s)                 Selector            Replicas
----------           ----------               ----------          ----------
frontendController   brendanburns/php-redis   name=frontend       3
```

Podsの作成までしばらく待つと、StatusがすべてRunningになります。

```
$ kubecfg list pods
ID                                     Image(s)                   Host                Labels                                                       Status
----------                             ----------                 ----------          ----------                                                   ----------
redis-master-2                         dockerfile/redis           10.1.0.207/         name=redis-master                                            Running
3c966f14-4984-11e4-8da7-02006edf0150   brendanburns/redis-slave   10.1.1.99/          name=redisslave,replicationController=redisSlaveController   Running
3c96c921-4984-11e4-8da7-02006edf0150   brendanburns/redis-slave   10.1.3.229/         name=redisslave,replicationController=redisSlaveController   Running
550828dd-4984-11e4-8da7-02006edf0150   brendanburns/php-redis     10.1.1.99/          name=frontend,replicationController=frontendController       Running
55085134-4984-11e4-8da7-02006edf0150   brendanburns/php-redis     10.1.0.207/         name=frontend,replicationController=frontendController       Running
55086ccf-4984-11e4-8da7-02006edf0150   brendanburns/php-redis     10.1.3.229/         name=frontend,replicationController=frontendController       Running
```

### 確認

* http://10.1.1.99:8000
* http://10.1.0.207:8000
* http://10.1.3.229:8000

{% img center /2014/10/02/kubernetes-fleet-flannel-guestbook/kube-fleet-flannel.png %}


