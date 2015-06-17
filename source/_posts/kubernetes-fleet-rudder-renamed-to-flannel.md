title: "Kubernetes on CoreOS with Fleet and flannel on IDCFクラウド - Part6: Rudder renamed to flannel"
date: 2014-09-27 02:24:41
tags:
 - Docker管理
 - Kubernetes
 - CoreOS
 - fleet
 - flannel
 - SDN
 - IDCFクラウド

description: Rudderのプロジェクト名がflannelに変わりました。すでに同じ名前のプロジェクトが存在しているからのようです。環境変数やバイナリ、ディレクトリなど修正する必要があります。この前つくったfleetクラスタの復習も兼ねてk8sクラスタを作り直すことにします。
---

* `Update 2014-10-02`: [Kubernetes on CoreOS with Fleet and flannel on IDCFクラウド - Part7: flannel版GuestBook](/2014/10/02/kubernetes-fleet-flannel-guestbook/)

Rudderのプロジェクト名が[flannel](https://github.com/coreos/flannel)に[変わりました](https://groups.google.com/forum/#!topic/coreos-user/usueEDHR3uQ)。すでに[同じ名前](http://www.rudder-project.org/site/)のプロジェクトが存在しているからのようです。環境変数やバイナリ、ディレクトリなど修正する必要があります。

この前つくった[fleetクラスタ](/2014/09/22/kubernetes-fleet-rudder-on-idcf/)の復習も兼ねてk8sクラスタを作り直すことにします。

<!-- more -->

### etcdのインスタンス

[前回](/2014/09/22/kubernetes-fleet-rudder-on-idcf/)と同様に、CoreOSは444.0.0のalphaのISOを使いインスタンスを作成します。

[kubernetes-fleet-tutorial](https://github.com/kelseyhightower/kubernetes-fleet-tutorial)の[etcd.yml](https://github.com/kelseyhightower/kubernetes-fleet-tutorial/blob/master/configs/etcd.yml)を参考にcloud-config.ymlを編集します。

* IPアドレスが`10.1.3.124`のところを、IDCFクラウドでインスタンスにアサインされた`10.1.3.124`に変更
* static.networkのunitsを省略
* 公開鍵を変更

``` yml ~/cloud-config.yml
#cloud-config

hostname: etcd
coreos:
  fleet:
    etcd_servers: http://127.0.0.1:4001
    metadata: role=etcd
  etcd:
    name: etcd
    addr: 10.1.3.124:4001
    bind-addr: 0.0.0.0
    peer-addr: 10.1.3.124:7001
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
    - ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDaMqAk6OeBLkyU6kj9HHq18xIBgSewSzZDRT3dYRnE92F5tz/16W/McxQ3XoIu8Z7aDNt5WEE+vWatAAItUPWQLRKjM0zwPC04hciMXloC7WjEeIikksg9hGzuc6nals7sfItCOx1aKqLFnEpRmXnzmKycYT8YqRBFzbGFJ+RSGmskVJeMaKUXunF43JH93ugmMCFqv92OnUHsY0tNeqI/EEncghHIKRwoHq46dNvECsvhX8ZruscQB3oGK24+sjZlL78kXeFXM4q3ZbefZESF4iKhs3BMfmT7ihNnuntdZhAHupBxg5npolw6yGZLjZcilduSnuTR6BDWtNEIS0S1 deis
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
$ ssh -A core@10.1.3.124
CoreOS (alpha)
```

### 作業マシンのfleetとetcd

作業マシンからetcdクラスタに接続するために環境変数を設定します。

``` bash
$ export ETCDCTL_PEERS=http://10.1.3.124:4001
$ export FLEETCTL_ENDPOINT=http://10.1.3.124:4001
```

fleetctlの確認です。

``` bash
$ fleetctl version
fleetctl version 0.8.1
$ fleetctl list-machines
MACHINE         IP              METADATA
1ee3b79b...     10.1.3.124      role=etcd
```

etcdctlの確認です。

```
$ etcdctl --version
etcdctl version 0.4.6
$ etcdctl ls /
```

### flannelの設定

etcdctlコマンドを使い、flannelで使用するネットワークを設定します。

``` bash
$ etcdctl mk /coreos.com/network/config '{"Network":"10.0.0.0/16"}'
{"Network":"10.0.0.0/16"}
```

### fleetクラスタの作成

Kubernetes用ノードは3台用意します。[node.yml](https://github.com/kelseyhightower/kubernetes-fleet-tutorial/blob/master/configs/node.yml)を参考にcloud-config.ymlを作成します。
この前rudderだった箇所がflannelに変更になっています。

* `etcd-endpoint`を作成したetcdサーバーのIPアドレス`10.1.3.124`を指定します。
* 公開鍵を変更

``` yaml ~/cloud-config.yml
#cloud-config

coreos:
  fleet:
    etcd_servers: http://10.1.3.124:4001
    metadata: role=kubernetes
  units:
    - name: etcd.service
      mask: true
    - name: fleet.service
      command: start
    - name: flannel.service
      command: start
      content: |
        [Unit]
        After=network-online.target 
        Wants=network-online.target
        Description=flannel is an etcd backed overlay network for containers

        [Service]
        ExecStartPre=-/usr/bin/mkdir -p /opt/bin
        ExecStartPre=/usr/bin/wget -N -P /opt/bin http://storage.googleapis.com/flannel/flanneld
        ExecStartPre=/usr/bin/chmod +x /opt/bin/flanneld
        ExecStart=/opt/bin/flanneld -etcd-endpoint http://10.1.3.124:4001
    - name: docker.service
      command: start
      content: |
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
    - ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDaMqAk6OeBLkyU6kj9HHq18xIBgSewSzZDRT3dYRnE92F5tz/16W/McxQ3XoIu8Z7aDNt5WEE+vWatAAItUPWQLRKjM0zwPC04hciMXloC7WjEeIikksg9hGzuc6nals7sfItCOx1aKqLFnEpRmXnzmKycYT8YqRBFzbGFJ+RSGmskVJeMaKUXunF43JH93ugmMCFqv92OnUHsY0tNeqI/EEncghHIKRwoHq46dNvECsvhX8ZruscQB3oGK24+sjZlL78kXeFXM4q3ZbefZESF4iKhs3BMfmT7ihNnuntdZhAHupBxg5npolw6yGZLjZcilduSnuTR6BDWtNEIS0S1 deis
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
$ ssh -A core@10.1.1.99
```

同様に合計で3台のKubernetes用インスタンスを作成します。

### 作業マシンからfleetクラスタを確認

fleetのクラスタを確認します。1台のetcdサーバーと3台のKubernetesノードを構築しました。

``` bash
$ export ETCDCTL_PEERS=http://10.1.3.124:4001
$ export FLEETCTL_ENDPOINT=http://10.1.3.124:4001
$ fleetctl list-machines
MACHINE         IP              METADATA
1ee3b79b...     10.1.3.124      role=etcd
2f706a2d...     10.1.1.99       role=kubernetes
6d444a3b...     10.1.3.229      role=kubernetes
ad6fa03e...     10.1.0.207      role=kubernetes
```

### Kubernetesのunitsファイルを用意

作業マシンでunitsファイルを`git clone`します。

``` bash
$ git clone https://github.com/kelseyhightower/kubernetes-fleet-tutorial.git
$ cd ~/kubernetes-fleet-tutorial/units
```

unitファイルの中でetcd_serversのIPアドレスをしている箇所を、実際に作成したetcdサーバーのIPアドレスに変更します。

* `etcd-endpoint`を作成したetcdサーバーのIPアドレス`10.1.3.124`を指定します。

kube-proxy.service

``` bash ~/kubernetes-fleet-tutorial/units/kube-proxy.service
[Unit]
Description=Kubernetes Proxy
Documentation=https://github.com/GoogleCloudPlatform/kubernetes

[Service]
ExecStartPre=/usr/bin/wget -N -P /opt/bin http://storage.googleapis.com/kubernetes/proxy
ExecStartPre=/usr/bin/chmod +x /opt/bin/proxy
ExecStart=/opt/bin/proxy --etcd_servers=http://10.1.3.124:4001 --logtostderr=true
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
--etcd_servers=http://10.1.3.124:4001 \
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
--etcd_servers=http://10.1.3.124:4001 \
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
$ export ETCDCTL_PEERS=http://10.1.3.124:4001
$ export FLEETCTL_ENDPOINT=http://10.1.3.124:4001
$ fleetctl start kube-proxy.service
Triggered global unit kube-proxy.service start
$ fleetctl start kube-kubelet.service
Triggered global unit kube-kubelet.service start
```

apiserver、scheduler、controller-managerは、Masterへインストールします。

``` bash
$ fleetctl start kube-apiserver.service
Unit kube-apiserver.service launched on 2f706a2d.../10.1.1.99
$ fleetctl start kube-scheduler.service
Unit kube-scheduler.service launched on 2f706a2d.../10.1.1.99
$ fleetctl start kube-controller-manager.service
Unit kube-controller-manager.service launched on 2f706a2d.../10.1.1.99
```

`list-units`で確認します。

``` bash
$ fleetctl list-units
UNIT                            MACHINE                 ACTIVE  SUB
kube-apiserver.service          2f706a2d.../10.1.1.99   active  running
kube-controller-manager.service 2f706a2d.../10.1.1.99   active  running
kube-kubelet.service            2f706a2d.../10.1.1.99   active  running
kube-kubelet.service            6d444a3b.../10.1.3.229  active  running
kube-kubelet.service            ad6fa03e.../10.1.0.207  active  running
kube-proxy.service              2f706a2d.../10.1.1.99   active  running
kube-proxy.service              6d444a3b.../10.1.3.229  active  running
kube-proxy.service              ad6fa03e.../10.1.0.207  active  running
kube-scheduler.service          2f706a2d.../10.1.1.99   active  running
```

### kube-register

[kube-register](https://github.com/kelseyhightower/kube-register)は、fleetのデータを使いKubelet マシンを登録できる便利なツールです。
これまで作成したunitsはfleetクラスタの世界なので、k8sクラスタとしては認識されていません。

kube-apiserver の環境変数`KUBERNETES_MASTER`を作業マシンに設定します。

``` bash
$ export KUBERNETES_MASTER="http://10.1.1.99:8080"
```

kube-registerを使い`Kubernetes API Server` に新しいノードの追加を通知します。
 
``` bash
$ cd ~/kubernetes-fleet-tutorial/units
$ fleetctl start kube-register.service
Unit kube-register.service launched on 2f706a2d.../10.1.1.99
```

### 作業マシンにkubecfgのインストール

作業マシンにkubecfgをインストールします。

``` bash
$ mkdir ~/temp
$ wget https://storage.googleapis.com/kubernetes/binaries.tar.gz
$ tar -xvf binaries.tar.gz -C ~/temp
$ sudo cp ~/temp/kubecfg /usr/local/bin
```

k8sクラスタとして認識されたので、fleetのnodesがkubecfgのminionとしてリストできるようになりました。

``` bash
$ kubecfg --version
Kubernetes v0.2-146-gc47dca5dbb9371
$ kubecfg list /minions
Minion identifier
----------
10.1.0.207
10.1.1.99
10.1.3.229
```

Podsはまだ作成していないのでリストされません。

``` bash
$ kubecfg list /pods
ID                  Image(s)            Host                Labels              Status
----------          ----------          ----------          ----------          ----------
```

### まとめ

だいぶ慣れてきたのでfleetクラスタとk8sクラスタの構築も、flannelのインストールも問題なくできるようになりました。
ようやく[GuestBook example](https://github.com/GoogleCloudPlatform/kubernetes/tree/master/examples/guestbook)を新しいクラスタにデプロイできるようになりました。

