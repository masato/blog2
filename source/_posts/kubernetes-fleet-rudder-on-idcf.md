title: "Kubernetes on CoreOS with Fleet and Rudder on IDCFクラウド - Part5: FleetとRudderで再作成する"
date: 2014-09-22 12:25:32
tags:
 - Docker管理
 - Kubernetes
 - CoreOS
 - fleet
 - Rudder
 - SDN
 - IDCFクラウド
description: 9月初めにIDCFクラウド上にRudderを使いCoreOSクラスタ上にインストールしたKubernetesを一度削除してから再作成しました。GuestBook exampleも同じ方法で作り直してデプロイしたのですが動かなくなりました。Google Groupsの同じ現象の投稿によると、new pods stuck in "waiting" status possbile cause Unable to parse docker config file unexpected end of JSON input Kubernetesのアーキテクチャに変更が入り、MasterコンポーネントにSchedulerが追加されました。以前作ったcloud-configにはSchedulerを追加していないのでMasterが動作しなくなったようです。
---

* `Update 2014-10-02`: [Kubernetes on CoreOS with Fleet and flannel on IDCFクラウド - Part7: flannel版GuestBook](/2014/10/02/kubernetes-fleet-flannel-guestbook/)
* `Update 2014-09-27`: [Kubernetes on CoreOS with Fleet and flannel on IDCFクラウド - Part6: Rudder renamed to flannel](/2014/09/27/kubernetes-fleet-rudder-renamed-to-flannel/)


9月初めにIDCFクラウド上に[Rudderを使いCoreOSクラスタ上にインストールしたKubernetes](/2014/09/04/kubernetes-in-coreos-on-idcf-with-rudder/)を一度削除してから再作成しました。[GuestBook example](/2014/09/05/kubernetes-in-coreos-on-idcf-guestbook-example/)も同じ方法で作り直してデプロイしたのですが動かなくなりました。

`Google Groups`の同じ現象の投稿によると、[new pods stuck in "waiting" status (possbile cause: Unable to parse docker config file: unexpected end of JSON input)](https://groups.google.com/forum/#!topic/google-containers/3r_cSKhk82c)
Kubernetesのアーキテクチャに変更が入り、MasterコンポーネントにSchedulerが追加されました。
以前作ったcloud-configにはSchedulerを追加していないのでMasterが動作しなくなったようです。

<!-- more -->

### Deploying Kubernetes on CoreOS with Fleet and Rudder

参考にしている[kubernetes-coreos](https://github.com/kelseyhightower/kubernetes-coreos)にはSchedulerの追加がまだありません。自分でScheulerのunitを書いたり、手動でSchedulerを開始したりすると少しずつ動くようになってきたところ、ちょうど良いサンプルが公開されました。

[kubernetes-fleet-tutorial](https://github.com/kelseyhightower/kubernetes-fleet-tutorial)というリポジトリではFleetでKubernetes用クラスタを構築しています。cloud-configやunitsファイルもとてもわかりやすくなりました。

さっそくFleetを使いIDCFクラウド上にKubernetesを構築していきます。だんだんとCoreOSっぽい使い方になってきました。

### CoreOS 444.0.0のISOダウンロード

[kubernetes-fleet-tutorial](https://github.com/kelseyhightower/kubernetes-fleet-tutorial)のCoreOSの要件は`v440.0.0+`です。2014-09-22の[Alpha Channelは444.0.0](https://coreos.com/docs/running-coreos/platforms/iso/)なのでalphaのISOを使います。

[ISOのURL](http://alpha.release.core-os.net/amd64-usr/current/coreos_production_iso_image.iso)をIDCFクラウドに登録して、新しいインスタンスを作成します。

### etcdインスタンス

これまでCorOSのクラスタとetcdのクラスタは同居して使ってきました。
kubernetes-fleet-tutorialでは専用のetcdクラスタを使うことを薦めています。CoreOSのbootstrapに失敗することがあるので今後もこのパターンで行きたいと思います。

ISOからインスタンス起動後、最初にコンソールから一時的にパスワード認証を許可します。

``` bash
$ sudo passwd core
```

作業マシンからパスワードでSSH接続します。

``` bash
$ ssh core@10.1.1.245
```

[etcd.yml](https://github.com/kelseyhightower/kubernetes-fleet-tutorial/blob/master/configs/etcd.yml)を参考にしてcloud-config.ymlを作成します。

IPアドレスを確認します。

``` bash
$ ip a
2: ens32: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP qlen 1000
    link/ether 02:00:17:cb:01:4a brd ff:ff:ff:ff:ff:ff
    inet 10.1.1.245/22 brd 10.1.3.255 scope global ens32
       valid_lft forever preferred_lft forever
    inet6 fe80::17ff:fecb:14a/64 scope link
       valid_lft forever preferred_lft forever
```

etcdクラスタは固定IPアドレスを指定します。この環境では`10.1.1.245`を使います。
`ssh_authorized_keys`はk8sクラスタで利用する公開鍵を記述します。

``` yml ~/cloud-config.yml
#cloud-config

hostname: etcd
coreos:
  fleet:
    etcd_servers: http://127.0.0.1:4001
    metadata: role=etcd
  etcd:
    name: etcd
    addr: 10.1.1.245:4001
    bind-addr: 0.0.0.0
    peer-addr: 10.1.1.245:7001
    cluster-active-size: 1
    snapshot: true
  units:
    - name: etcd.service
      command: start
    - name: fleet.service
      command: start
    - name: docker.service
      mask: true
  update:
    group: alpha
    reboot-strategy: off
ssh_authorized_keys:
    - ssh-rsa AAAAB3Nz...
```

`CoreOS alpha 444.0.0`から`/dev/sda`にCoreOSをインストールします。

```
$ sudo coreos-install -d /dev/sda  -C alpha -c ./cloud-config.yml
...
Installing cloud-config...
Success! CoreOS alpha 444.0.0 is installed on /dev/sda
```

rebootして、公開鍵認証でSSH接続の確認をします。

``` bash
$ eval `ssh-agent`
$ ssh-add ~/.ssh/deis
$ ssh -A core@10.1.1.245
CoreOS (alpha)
```

### 作業マシンのfleetとetcd

作業マシンからetcdクラスタに接続するために環境変数を設定します。

``` bash
$ export ETCDCTL_PEERS=http://10.1.1.245:4001
$ export FLEETCTL_ENDPOINT=http://10.1.1.245:4001
```

fleetctlを実行するとバージョンが古いため警告がでます。

``` bash
$ fleetctl list-machines
####################################################################
WARNING: fleetctl (0.7.1+git) is older than the latest registered
version of fleet found in the cluster (0.8.1). You are strongly
recommended to upgrade fleetctl to prevent incompatibility issues.
####################################################################
```

作業環境もクラスタのfleetとバージョンを合わせるため、0.8.1をインストールします。

``` bash
$ wget https://github.com/coreos/fleet/releases/download/v0.8.1/fleet-v0.8.1-linux-amd64.tar.gz
$ tar zxvf fleet-v0.8.1-linux-amd64.tar.gz
fleet-v0.8.1-linux-amd64/
fleet-v0.8.1-linux-amd64/fleetctl
fleet-v0.8.1-linux-amd64/fleetd
fleet-v0.8.1-linux-amd64/README.md
$ sudo cp fleet-v0.8.1-linux-amd64/fleet* /usr/local/bin/
$ fleetctl version
fleetctl version 0.8.1
```

etcd v0.4.6をインストールします。

``` bash
$ wget https://github.com/coreos/etcd/releases/download/v0.4.6/etcd-v0.4.6-linux-amd64.tar.gz
$ tar zxvf etcd-v0.4.6-linux-amd64.tar.gz
etcd-v0.4.6-linux-amd64/
etcd-v0.4.6-linux-amd64/etcd
etcd-v0.4.6-linux-amd64/etcdctl
etcd-v0.4.6-linux-amd64/README-etcd.md
etcd-v0.4.6-linux-amd64/README-etcdctl.md
$ sudo cp etcd-v0.4.6-linux-amd64/etcd* /usr/local/bin/
$ etcdctl ls /
```

### Rudderの設定

etcdctlコマンドを使い、Rudderで使用するネットワークを設定します。

``` bash
$ etcdctl mk /coreos.com/network/config '{"Network":"10.0.0.0/16"}'
{"Network":"10.0.0.0/16"}
```


### Kubernetes用ノード作成

Kubernetes用ノードは3台用意します。etcdインスタンスと同様に`CoreOS alpha 444.0.0`をISOからインスタンスを作成します。
[node.yml](https://github.com/kelseyhightower/kubernetes-fleettutorial/blob/master/configs/node.yml)を参考にcloud-config.ymlを作成します。`etcd-endpoint`に作成したetcdサーバーのIPアドレスを指定します。

``` yaml ~/cloud-config.yml
#cloud-config

coreos:
  fleet:
    etcd_servers: http://10.1.1.245:4001
    metadata: role=kubernetes
  units:
    - name: etcd.service
      mask: true
    - name: fleet.service
      command: start
    - name: rudder.service
      command: start
      content: |
        [Unit]
        After=network-online.target 
        Wants=network-online.target
        Description=Rudder is an etcd backed overlay network for containers

        [Service]
        ExecStartPre=-/usr/bin/mkdir -p /opt/bin
        ExecStartPre=/usr/bin/wget -N -P /opt/bin https://storage.googleapis.com/rudder/rudder
        ExecStartPre=/usr/bin/chmod +x /opt/bin/rudder
        ExecStart=/opt/bin/rudder -etcd-endpoint http://10.1.1.245:4001
    - name: docker.service
      command: start
      content: |
        [Unit]
        After=rudder.service
        Wants=rudder.service
        Description=Docker Application Container Engine
        Documentation=http://docs.docker.io

        [Service]
        EnvironmentFile=/run/rudder/subnet.env
        ExecStartPre=/bin/mount --make-rprivate /
        ExecStart=/usr/bin/docker -d --bip=${RUDDER_SUBNET} --mtu=${RUDDER_MTU} -s=btrfs -H fd:// --iptables=false

        [Install]
        WantedBy=multi-user.target
    - name: setup-network-environment.service
      command: start
      content: |
        [Unit]
        Description=Setup Network Environment
        Documentation=https://github.com/kelseyhightower/setup-network-environment
        Requires=network-online.target
        After=network-online.target

        [Service]
        ExecStartPre=-/usr/bin/mkdir -p /opt/bin
        ExecStartPre=/usr/bin/wget -N -P /opt/bin http://storage.googleapis.com/snenv/setup-network-environment 
        ExecStartPre=/usr/bin/chmod +x /opt/bin/setup-network-environment
        ExecStart=/opt/bin/setup-network-environment
        RemainAfterExit=yes
        Type=oneshot
  update:
    group: alpha
    reboot-strategy: off
ssh_authorized_keys:
    - ssh-rsa AAAAB3Nz...
```

`CoreOS alpha 444.0.0`から`/dev/sda`にCoreOSをインストールします。

``` bash
$ sudo coreos-install -d /dev/sda  -C alpha -c ./cloud-config.yml
...
Installing cloud-config...
Success! CoreOS alpha 444.0.0 is installed on /dev/sda
```

reboot後にSSH公開鍵でログインできます。

``` bash
$ eval `ssh-agent`
$ ssh-add ~/.ssh/deis
$ ssh -A core@10.1.0.77
```

### 作業マシンからfleetクラスタを確認

作業マシンから、fleetのクラスタを確認します。1台のetcdサーバーと3台のKubernetesノードを構築しました。

``` bash
$ export ETCDCTL_PEERS=http://10.1.1.245:4001
$ export FLEETCTL_ENDPOINT=http://10.1.1.245:4001
$ fleetctl list-machines
MACHINE         IP              METADATA
07914a50...     10.1.0.77       role=kubernetes
46bdac0c...     10.1.1.180      role=kubernetes
bb805b88...     10.1.1.27       role=kubernetes
c31a7001...     10.1.1.245      role=etcd
```

### Kubernetesのunitsファイルを用意

作業マシンでunitsファイルを`git clone`します。

``` bash
$ git clone https://github.com/kelseyhightower/kubernetes-fleet-tutorial.git
$ cd ~/kubernetes-fleet-tutorial/units
```

unitファイルの中でetcd_serversのIPアドレスをしている箇所を、実際に作成したetcdサーバーのIPアドレスに変更します。

kube-proxy.service

``` bash ~/kubernetes-fleet-tutorial/units/kube-proxy.service
[Unit]
Description=Kubernetes Proxy
Documentation=https://github.com/GoogleCloudPlatform/kubernetes

[Service]
ExecStartPre=/usr/bin/wget -N -P /opt/bin http://storage.googleapis.com/kubernetes/proxy
ExecStartPre=/usr/bin/chmod +x /opt/bin/proxy
ExecStart=/opt/bin/proxy --etcd_servers=http://10.1.1.245:4001 --logtostderr=true
Restart=always
RestartSec=10

[X-Fleet]
Global=true
MachineMetadata=role=kubernetes
```


kube-kubelet.service

``` bash ~/kubernetes-fleet-tutorial/units/kube-kubelet.service
[Unit]
Description=Kubernetes Kubelet
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
Requires=setup-network-environment.service
After=setup-network-environment.service

[Service]
EnvironmentFile=/etc/network-environment
ExecStartPre=/usr/bin/wget -N -P /opt/bin http://storage.googleapis.com/kubernetes/kubelet
ExecStartPre=/usr/bin/chmod +x /opt/bin/kubelet
ExecStart=/opt/bin/kubelet \
--address=0.0.0.0 \
--port=10250 \
--hostname_override=${DEFAULT_IPV4} \
--etcd_servers=http://10.1.1.245:4001 \
--logtostderr=true
Restart=always
RestartSec=10

[X-Fleet]
Global=true
MachineMetadata=role=kubernetes
```

kube-apiserver.service

``` bash ~/kubernetes-fleet-tutorial/units/kube-apiserver.service
[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/GoogleCloudPlatform/kubernetes

[Service]
ExecStartPre=/usr/bin/wget -N -P /opt/bin http://storage.googleapis.com/kubernetes/apiserver
ExecStartPre=/usr/bin/chmod +x /opt/bin/apiserver
ExecStart=/opt/bin/apiserver \
--address=0.0.0.0 \
--port=8080 \
--etcd_servers=http://10.1.1.245:4001 \
--logtostderr=true
Restart=always
RestartSec=10

[X-Fleet]
MachineMetadata=role=kubernetes
```

### Kubernetesのインストール

proxyとkubeletのunitsは、すべてのKubernetesノードにインストールします。
`[X-Fleet]`に、`Global=true`となっているunitsは、fleetクラスタの全ノードにインストールされます。

``` bash
$ fleetctl start kube-proxy.service
Triggered global unit kube-proxy.service start
$ fleetctl start kube-kubelet.service
Triggered global unit kube-kubelet.service start
```

`list-units`で確認します。

``` bash
$ fleetctl list-units
UNIT                    MACHINE                 ACTIVE  SUB
kube-kubelet.service    07914a50.../10.1.0.77   active  running
kube-kubelet.service    46bdac0c.../10.1.1.180  active  running
kube-kubelet.service    bb805b88.../10.1.1.27   active  running
kube-proxy.service      07914a50.../10.1.0.77   active  running
kube-proxy.service      46bdac0c.../10.1.1.180  active  running
kube-proxy.service      bb805b88.../10.1.1.27   active  running
```

apiserver、scheduler、controller-managerは、Masterへインストールします。

``` bash
$ fleetctl start kube-apiserver.service
Unit kube-apiserver.service launched on 07914a50.../10.1.0.77
$ fleetctl start kube-scheduler.service
Unit kube-scheduler.service launched on 07914a50.../10.1.0.77
$ fleetctl start kube-controller-manager.service
Unit kube-controller-manager.service launched on 07914a50.../10.1.0.77
```

`list-units`で確認します。

``` bash
$ export ETCDCTL_PEERS=http://10.1.1.245:4001
$ export FLEETCTL_ENDPOINT=http://10.1.1.245:4001
$ fleetctl list-units
UNIT                            MACHINE                 ACTIVE  SUB
kube-apiserver.service          07914a50.../10.1.0.77   active  running
kube-controller-manager.service 07914a50.../10.1.0.77   active  running
kube-kubelet.service            07914a50.../10.1.0.77   active  running
kube-kubelet.service            46bdac0c.../10.1.1.180  active  running
kube-kubelet.service            bb805b88.../10.1.1.27   active  running
kube-proxy.service              07914a50.../10.1.0.77   active  running
kube-proxy.service              46bdac0c.../10.1.1.180  active  running
kube-proxy.service              bb805b88.../10.1.1.27   active  running
kube-scheduler.service          07914a50.../10.1.0.77   active  running
```

### kube-register

[kube-register](https://github.com/kelseyhightower/kube-register)は、fleetのデータを使いKubelet マシンを登録できる便利なツールです。
これまで作成したunitsはfleetクラスタの世界なので、k8sクラスタとしては認識されていません。

kube-apiserver の環境変数`KUBERNETES_MASTER`を作業マシンに設定します。

``` bash
$ export KUBERNETES_MASTER="http://10.1.0.77:8080"
```

kube-registerを使い`Kubernetes API Server` に新しいノードの追加を通知します。
 
``` bash
$ cd ~/kubernetes-fleet-tutorial/units
$ fleetctl start kube-register.service
Unit kube-register.service launched on 07914a50.../10.1.0.77
```

kubecfg list /minions


### 作業マシンにkubecfgのインストール

作業マシンから`Kubernetes API Server`にSSHトンネルを作成します。

``` bash
$ eval `ssh-agent`
$ ssh-add ~/.ssh/deis
$ ssh -f -nNT -L 8080:127.0.0.1:8080 core@10.1.0.77
$ curl -s http://localhost:8080
<html><body>Welcome to Kubernetes</body></html>
```

作業マシンにkubecfgをインストールします。

``` bash
$ mkdir ~/temp
$ wget https://storage.googleapis.com/kubernetes/binaries.tar.gz
$ tar -xvf binaries.tar.gz -C ~/temp
$ sudo cp ~/temp/kubecfg /usr/local/bin
```


k8sクラスタとして認識されたので、fleetのnodesがkubecfgのminionとしてリストできるようになりました。

``` bash
$ kubecfg list /minions
Minion identifier
----------
10.1.0.77
10.1.1.180
10.1.1.27
```

Podsはまだ作成していないのでリストされません。

``` bash
$ kubecfg list /pods
ID                  Image(s)            Host                Labels              Status
----------          ----------          ----------          ----------          ----------
```

### まとめ

壊れてしまったKubernetesクラスタが新しくFleetとRudderを使い構築できました。
今回からFleet経由になったので、よりCoreOSらしくなり構造もわかりやすくなりました。

次は[GuestBook example](https://github.com/GoogleCloudPlatform/kubernetes/tree/master/examples/guestbook)のサンプルを使いPodsを作成してみます。
