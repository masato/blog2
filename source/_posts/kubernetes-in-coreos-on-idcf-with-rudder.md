title: "Kubernetes in CoreOS with Rudder on IDCFクラウド - Part3: RudderでSDN風なこと"
date: 2014-09-04 00:42:32
tags:
 - Docker管理
 - Kubernetes
 - CoreOS
 - cloud-config
 - etcd
 - Rudder
 - SDN
 - IDCFクラウド
description: Kubernetesをスタンドアロンでマニュアルインストールはできたのですが、GCEやAzureのようにインストールスクリプトが用意されていないIDCFクラウドではどうすればいいか考えていました。kubernetes-coreosに、CoreOSクラスタをインストールするcloud-configはありますが、fleet(systemd)とネットワーク設定がよく理解できません。cbr0のネットワークデバイスの作成とiptablesはムズカシイ。Introducing Rudder)という記事をみつけ、Rudderを使うとネットワークとサブネットを自由に作れないIDCFクラウドでもSDNっぽいことができそうです。Docker & Kubernetes は SDN のカタリストになれるかも知れないという記事でDockerとKubernetesが広げる、ネットワークとストレージのコンテナへの融合は、これからのクラウドが目指す方向だと感じます。KubernetesのNetworkingを読むと、PodとNATレスなネットワークの重要性について書いてあり、Googleデータセンターの片鱗が見えます。本当に`tip of the iceberg`なのですが。SDNはまだ数年先で、ネットワークエンジニアが扱うもの思っていました。KubernetesとRudderを使うと簡単にSDNっぽく作れます。OVS等使っていないのですが、etcdのおかげプログラマでもとてもカジュアルに使えます。
---

* `Update 2014-10-02`: [Kubernetes on CoreOS with Fleet and flannel on IDCFクラウド - Part7: flannel版GuestBook](/2014/10/02/kubernetes-fleet-flannel-guestbook/)
* `Update 2014-09-27`: [Kubernetes on CoreOS with Fleet and flannel on IDCFクラウド - Part6: Rudder renamed to flannel](/2014/09/27/kubernetes-fleet-rudder-renamed-to-flannel/)
* `Update 2014-09-22`: [Kubernetes on CoreOS with Fleet and Rudder on IDCFクラウド - Part5: FleetとRudderで再作成する](/2014/09/22/kubernetes-fleet-rudder-on-idcf/)
* `Update 2014-09-05`: [Kubernetes in CoreOS with Rudder on IDCFクラウド - Part4: GuestBook example](/2014/09/05/kubernetes-in-coreos-on-idcf-guestbook-example/)

[Kubernetesをスタンドアロンでマニュアルインストール](/2014/08/31/kubernetes-on-coreos-cluster/)はできたのですが、GCEやAzureのようにインストールスクリプトが用意されていないIDCFクラウドではどうすればいいか考えていました。
[kubernetes-coreos](https://github.com/kelseyhightower/kubernetes-coreos)に、CoreOSクラスタをインストールするcloud-configはありますが、fleet(systemd)とネットワーク設定がよく理解できません。cbr0のネットワークデバイスの作成とiptablesはムズカシイ。

[Introducing Rudder](https://coreos.com/blog/introducing-rudder/)という記事をみつけ、Rudderを使うとネットワークとサブネットを自由に作れないIDCFクラウドでもSDNっぽいことができそうです。

[Docker & Kubernetes は SDN のカタリストになれるかも知れない](https://www.sdncentral.com/japanese/docker-kubernetes-%E3%81%AF-sdn-%E3%81%AE%E3%82%AB%E3%82%BF%E3%83%AA%E3%82%B9%E3%83%88%E3%81%AB%E3%81%AA%E3%82%8C%E3%82%8B%E3%81%8B%E3%82%82%E7%9F%A5%E3%82%8C%E3%81%AA%E3%81%84/2014/07/)という記事でDockerとKubernetesが広げる、ネットワークとストレージのコンテナへの融合は、これからのクラウドが目指す方向だと感じます。

Kubernetesの[Networking](https://github.com/GoogleCloudPlatform/kubernetes/blob/master/docs/networking.md)を読むと、PodとNATレスなネットワークの重要性について書いてあり、Googleデータセンターの片鱗が見えます。本当に`tip of the iceberg`なのですが。

SDNはまだ数年先で、ネットワークエンジニアが扱うもの思っていました。KubernetesとRudderを使うと簡単にSDNっぽく作れます。OVS等使っていないのですが、etcdのおかげプログラマでもとてもカジュアルに使えます。

<!-- more -->


### Rudderのインストール

RudderはCoreOSにインストールしますが、GoでビルドするのでUbuntu(LinuxMint17)上で行えます。

``` bash
$ sudo apt-get install linux-libc-dev
$ cd
$ git clone https://github.com/coreos/rudder.git
$ cd rudder; ./build
```

CoreOSのノード3台にSCPで転送します。

``` bash
$ eval `ssh-agent`
$ ssh-add ~/.ssh/deis
$ scp ~/rudder/bin/rudder core@10.1.1.140:
$ scp ~/rudder/bin/rudder core@10.1.3.13:
$ scp ~/rudder/bin/rudder core@10.1.0.44:
```

CoreOSのノード3台にログインして、PATHが通っている`/opt/bin`に配置します。

``` bash
$ sudo mkdir -p /opt/bin
$ sudo mv ./rudder /opt/bin/
$ sudo reboot
```

### fleetctlでCoreOSクラスタの確認

LinuxMint17のワークステーションから、CoreOSクラスタを確認します。

``` bash
$ export FLEETCTL_TUNNEL=10.1.0.44
$ fleetctl list-machines
MACHINE         IP              METADATA
4b3c0589...     10.1.0.44       -
f696f27e...     10.1.3.13       -
fdb23eca...     10.1.1.140      -
```

### masterノード

fleetctlでCoreOSクラスタの確認して、fdb23eca(10.1.1.140)のノードをmasterにします。

fdb23eca(10.1.1.140)のコンテナにSSH接続します。

``` bash
$ eval `ssh-agent`
$ ssh-add ~/.ssh/deis
$ fleetctl ssh fdb23eca
```

Rudderを使いGCEのdefaultネットワークに相当する、`10.240.0.0/16`のネットワークを作成します。


``` yaml
ExecStartPre=-/usr/bin/etcdctl mk /coreos.com/network/config '{"Network":"10.240.0.0/16"}'
```

[Using Cloud-Config](https://coreos.com/docs/cluster-management/setup/cloudinit-cloud-config/)にあるように、CoreOSのupdateは停止しておきます。

``` yaml
  update:
    group: stable
    reboot-strategy: off
```

コンテナのデバッグ用に、CoreOSのノードにSSH接続したあと、nsenterが使えるように`write_files:`しておきます。

``` yaml
  - path: /etc/profile.d/nse-function.sh
    permissions: 0755
    content: |
      function nse() {
        sudo nsenter --pid --uts --mount --ipc --net --target $(docker inspect --format="&#123;&#123; .State.Pid }}" $1)
      }
```      

`download-kubernetes.service`以下のunitは、[master.yml](https://github.com/kelseyhightower/kubernetes-coreos/blob/master/configs/master.yml)と同じです。

`user_data`を編集して、RudderとKubernetesのmasterとminionをインストールします。

``` yaml /var/lib/coreos-install/user_data
#cloud-config

ssh_authorized_keys:
  - ssh-rsa AAAAB3NzaC...

coreos:
  etcd:
    discovery: https://discovery.etcd.io/b9d070ef8fc3c65b3dc8a49853a77adf
    addr: 10.1.1.140:4001
    peer-addr: 10.1.1.140:7001
  units:
    - name: rudder.service
      command: start
      content: |
        [Unit]
        Requires=etcd.service
        After=etcd.service

        [Service]
        ExecStartPre=-/usr/bin/etcdctl mk /coreos.com/network/config '{"Network":"10.240.0.0/16"}'
        ExecStart=/opt/bin/rudder

        [Install]
        WantedBy=multi-user.target
    - name: docker.service
      command: restart
      content: |
        [Unit]
        Description=Docker Application Container Engine
        Documentation=http://docs.docker.io
        Requires=rudder.service
        After=rudder.service

        [Service]
        EnvironmentFile=/run/rudder/subnet.env
        ExecStartPre=/bin/mount --make-rprivate /
        ExecStart=/usr/bin/docker -d -s=btrfs -H fd:// --bip=${RUDDER_SUBNET} --mtu=${RUDDER_MTU}

        [Install]
        WantedBy=multi-user.target
    - name: download-kubernetes.service
      command: start
      content: |
        [Unit]
        After=network-online.target
        Before=apiserver.service
        Before=controller-manager.service
        Before=kubelet.service
        Before=proxy.service
        Description=Download Kubernetes Binaries
        Documentation=https://github.com/GoogleCloudPlatform/kubernetes
        Requires=network-online.target

        [Service]
        ExecStart=/usr/bin/wget -N -P /opt/bin http://storage.googleapis.com/kubernetes/apiserver
        ExecStart=/usr/bin/wget -N -P /opt/bin http://storage.googleapis.com/kubernetes/controller-manager
        ExecStart=/usr/bin/wget -N -P /opt/bin http://storage.googleapis.com/kubernetes/kubecfg
        ExecStart=/usr/bin/wget -N -P /opt/bin http://storage.googleapis.com/kubernetes/kubelet
        ExecStart=/usr/bin/wget -N -P /opt/bin http://storage.googleapis.com/kubernetes/proxy
        ExecStart=/usr/bin/chmod +x /opt/bin/apiserver
        ExecStart=/usr/bin/chmod +x /opt/bin/controller-manager
        ExecStart=/usr/bin/chmod +x /opt/bin/kubecfg
        ExecStart=/usr/bin/chmod +x /opt/bin/kubelet
        ExecStart=/usr/bin/chmod +x /opt/bin/proxy
        RemainAfterExit=yes
        Type=oneshot
    - name: apiserver.service
      command: start
      content: |
        [Unit]
        After=etcd.service
        After=download-kubernetes.service
        ConditionFileIsExecutable=/opt/bin/apiserver
        Description=Kubernetes API Server
        Documentation=https://github.com/GoogleCloudPlatform/kubernetes
        Wants=etcd.service
        Wants=download-kubernetes.service

        [Service]
        ExecStart=/opt/bin/apiserver \
        --address=127.0.0.1 \
        --port=8080 \
        --etcd_servers=http://127.0.0.1:4001 \
        --machines=10.1.1.140,10.1.3.13,10.1.0.44 \
        --logtostderr=true
        Restart=always
        RestartSec=10

        [Install]
        WantedBy=multi-user.target
    - name: controller-manager.service
      command: start
      content: |
        [Unit]
        After=etcd.service
        After=download-kubernetes.service
        ConditionFileIsExecutable=/opt/bin/controller-manager
        Description=Kubernetes Controller Manager
        Documentation=https://github.com/GoogleCloudPlatform/kubernetes
        Wants=etcd.service
        Wants=download-kubernetes.service

        [Service]
        ExecStart=/opt/bin/controller-manager \
        --master=127.0.0.1:8080 \
        --logtostderr=true
        Restart=always
        RestartSec=10

        [Install]
        WantedBy=multi-user.target
    - name: kubelet.service
      command: start
      content: |
        [Unit]
        After=etcd.service
        After=download-kubernetes.service
        ConditionFileIsExecutable=/opt/bin/kubelet
        Description=Kubernetes Kubelet
        Documentation=https://github.com/GoogleCloudPlatform/kubernetes
        Wants=etcd.service
        Wants=download-kubernetes.service

        [Service]
        ExecStart=/opt/bin/kubelet \
        --address=0.0.0.0 \
        --port=10250 \
        --hostname_override=10.1.1.140 \
        --etcd_servers=http://127.0.0.1:4001 \
        --logtostderr=true
        Restart=always
        RestartSec=10

        [Install]
        WantedBy=multi-user.target
    - name: proxy.service
      command: start
      content: |
        [Unit]
        After=etcd.service
        After=download-kubernetes.service
        ConditionFileIsExecutable=/opt/bin/proxy
        Description=Kubernetes Proxy
        Documentation=https://github.com/GoogleCloudPlatform/kubernetes
        Wants=etcd.service
        Wants=download-kubernetes.service

        [Service]
        ExecStart=/opt/bin/proxy --etcd_servers=http://127.0.0.1:4001 --logtostderr=true
        Restart=always
        RestartSec=10

        [Install]
        WantedBy=multi-user.target
    - name: etcd.service
      command: start
    - name: fleet.service
      command: start
  update:
    group: stable
    reboot-strategy: off
write_files:
  - path: /etc/environment
    permissions: 0644
    content: |
      COREOS_PUBLIC_IPV4=
      COREOS_PRIVATE_IPV4=10.1.1.140
  - path: /etc/profile.d/nse-function.sh
    permissions: 0755
    content: |
      function nse() {
        sudo nsenter --pid --uts --mount --ipc --net --target $(docker inspect --format="&#123;&#123; .State.Pid }}" $1)
      }
```

masterノードをrebootします。

``` bash
$ sudo reboot
```

masterノードにSSH接続して、設定されたネットワークを確認します。
Kubernetesがdocker0インタフェース用に、`10.240.26.0/16`のSDNから`10.240.26.1/24`のサブネットを作ってくれます。

``` bash
$ ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: ens32: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP qlen 1000
    link/ether 02:00:1f:3a:01:28 brd ff:ff:ff:ff:ff:ff
    inet 10.1.1.140/22 brd 10.1.3.255 scope global ens32
       valid_lft forever preferred_lft forever
    inet6 fe80::1fff:fe3a:128/64 scope link
       valid_lft forever preferred_lft forever
3: rudder0: <POINTOPOINT,UP,LOWER_UP> mtu 1472 qdisc pfifo_fast state UNKNOWN qlen 500
    link/none
    inet 10.240.26.0/16 scope global rudder0
       valid_lft forever preferred_lft forever
4: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN
    link/ether 56:84:7a:fe:97:99 brd ff:ff:ff:ff:ff:ff
    inet 10.240.26.1/24 scope global docker0
       valid_lft forever preferred_lft forever
```

routeを見てみます。10.240.0.0へのrudder0インタフェースができました。
docker0インタフェースもデフォルトの172-dotアドレス空間でなく、SDNの10-dotアドレス空間に向いています。

``` bash
$ route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         10.1.0.1        0.0.0.0         UG    0      0        0 ens32
10.1.0.0        0.0.0.0         255.255.252.0   U     0      0        0 ens32
10.1.0.1        0.0.0.0         255.255.255.255 UH    0      0        0 ens32
10.240.0.0      0.0.0.0         255.255.0.0     U     0      0        0 rudder0
10.240.26.0     0.0.0.0         255.255.255.0   U     0      0        0 docker0
```

Rudderがmasterノードに割り当てた、/24のサブネットを確認します。

``` bash
$ cat /run/rudder/subnet.env
RUDDER_SUBNET=10.240.26.1/24
RUDDER_MTU=1472
```

### minionノード

4b3c0589(10.1.0.44)とf696f27e(10.1.3.13)をminionノードにします。
IPアドレスの設定が違うだけで、`user_data`は同じです。


``` yaml /var/lib/coreos-install/user_data
#cloud-config

ssh_authorized_keys:
  - ssh-rsa AAAAB3NzaC...

coreos:
  etcd:
    discovery: https://discovery.etcd.io/b9d070ef8fc3c65b3dc8a49853a77adf
    addr: 10.1.3.13:4001
    peer-addr: 10.1.3.13:7001
  units:
    - name: rudder.service
      command: start
      content: |
        [Unit]
        Requires=etcd.service
        After=etcd.service

        [Service]
        ExecStartPre=-/usr/bin/etcdctl mk /coreos.com/network/config '{"Network":"10.240.0.0/16"}'
        ExecStart=/opt/bin/rudder

        [Install]
        WantedBy=multi-user.target
    - name: docker.service
      command: restart
      content: |
        [Unit]
        Description=Docker Application Container Engine
        Documentation=http://docs.docker.io
        Requires=rudder.service
        After=rudder.service

        [Service]
        EnvironmentFile=/run/rudder/subnet.env
        ExecStartPre=/bin/mount --make-rprivate /
        ExecStart=/usr/bin/docker -d -s=btrfs -H fd:// --bip=${RUDDER_SUBNET} --mtu=${RUDDER_MTU}

        [Install]
        WantedBy=multi-user.target
    - name: download-kubernetes.service
      command: start
      content: |
        [Unit]
        After=network-online.target
        Before=kubelet.service
        Before=proxy.service
        Description=Download Kubernetes Binaries
        Documentation=https://github.com/GoogleCloudPlatform/kubernetes
        Requires=network-online.target

        [Service]
        ExecStart=/usr/bin/wget -N -P /opt/bin http://storage.googleapis.com/kubernetes/kubelet
        ExecStart=/usr/bin/wget -N -P /opt/bin http://storage.googleapis.com/kubernetes/proxy
        ExecStart=/usr/bin/chmod +x /opt/bin/kubelet
        ExecStart=/usr/bin/chmod +x /opt/bin/proxy
        RemainAfterExit=yes
        Type=oneshot
    - name: kubelet.service
      command: start
      content: |
        [Unit]
        After=etcd.service
        After=download-kubernetes.service
        ConditionFileIsExecutable=/opt/bin/kubelet
        Description=Kubernetes Kubelet
        Documentation=https://github.com/GoogleCloudPlatform/kubernetes
        Wants=etcd.service
        Wants=download-kubernetes.service

        [Service]
        ExecStart=/opt/bin/kubelet \
        --address=0.0.0.0 \
        --port=10250 \
        --hostname_override=10.1.3.13 \
        --etcd_servers=http://127.0.0.1:4001 \
        --logtostderr=true
        Restart=always
        RestartSec=10

        [Install]
        WantedBy=multi-user.target
    - name: proxy.service
      command: start
      content: |
        [Unit]
        After=etcd.service
        After=download-kubernetes.service
        ConditionFileIsExecutable=/opt/bin/proxy
        Description=Kubernetes Proxy
        Documentation=https://github.com/GoogleCloudPlatform/kubernetes
        Wants=etcd.service
        Wants=download-kubernetes.service

        [Service]
        ExecStart=/opt/bin/proxy --etcd_servers=http://127.0.0.1:4001 --logtostderr=true
        Restart=always
        RestartSec=10

        [Install]
        WantedBy=multi-user.target
    - name: etcd.service
      command: start
    - name: fleet.service
      command: start
  update:
    group: stable
    reboot-strategy: off
write_files:
  - path: /etc/environment
    permissions: 0644
    content: |
      COREOS_PUBLIC_IPV4=
      COREOS_PRIVATE_IPV4=10.1.3.13
  - path: /etc/profile.d/nse-function.sh
    permissions: 0755
    content: |
      function nse() {
        sudo nsenter --pid --uts --mount --ipc --net --target $(docker inspect --format="&#123;&#123; .State.Pid }}" $1)
      }
```

minionノードをrebootします。

``` bash
$ sudo reboot
```

minionノードにSSH接続して、設定されたネットワークを確認します。

``` bash
 $ ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: ens32: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP qlen 1000
    link/ether 02:00:61:84:01:27 brd ff:ff:ff:ff:ff:ff
    inet 10.1.3.13/22 brd 10.1.3.255 scope global ens32
       valid_lft forever preferred_lft forever
    inet6 fe80::61ff:fe84:127/64 scope link
       valid_lft forever preferred_lft forever
3: rudder0: <POINTOPOINT,UP,LOWER_UP> mtu 1472 qdisc pfifo_fast state UNKNOWN qlen 500
    link/none
    inet 10.240.69.0/16 scope global rudder0
       valid_lft forever preferred_lft forever
4: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN
    link/ether 56:84:7a:fe:97:99 brd ff:ff:ff:ff:ff:ff
    inet 10.240.69.1/24 scope global docker0
       valid_lft forever preferred_lft forever
```


routeを見てみます。10.240.0.0へのrudder0インタフェースができました。

``` bash
$ route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         10.1.0.1        0.0.0.0         UG    0      0        0 ens32
10.1.0.0        0.0.0.0         255.255.252.0   U     0      0        0 ens32
10.1.0.1        0.0.0.0         255.255.255.255 UH    0      0        0 ens32
10.240.0.0      0.0.0.0         255.255.0.0     U     0      0        0 rudder0
10.240.69.0     0.0.0.0         255.255.255.0   U     0      0        0 docker0
```

Rudderがmasterノードに割り当てた、/24のサブネットを確認します。

``` bash
$ cat /run/rudder/subnet.env
RUDDER_SUBNET=10.240.69.1/24
RUDDER_MTU=1472
```

### ワークステーションから確認

LinuxMint17のワークステーションから、masterノードへSSHトンネルを作ります。

``` bash
$ eval `ssh-agent`
$ ssh-add ~/.ssh/deis
$ ssh -f -nNT -L 8080:127.0.0.1:8080 core@10.1.1.140
```

Kubernetesへ接続できることを確認します。

``` bash
$ curl -s http://localhost:8080
<html><body>Welcome to Kubernetes</body></html>
```

ワークステーションにkubecfgをインストールします。

``` bash
$ sudo wget https://storage.googleapis.com/kubernetes/kubecfg -P /usr/local/bin/
$ sudo chmod +x /usr/local/bin/kubecfg
```

Podはまだ作成していないので、リストされません。

``` bash
$ /usr/local/bin/kubecfg list /pods
Name                Image(s)            Host                Labels
----------          ----------          ----------          ----------
```

KubernetesのMinion3台がリストされました。

``` bash
$ /usr/local/bin/kubecfg list minions
Minion identifier
----------
10.1.0.44
10.1.1.140
10.1.3.13
```

### まとめ

ここまで苦労しましたが、ようやくIDCFクラウド上のCoreOSクラスタに待望のKubernetesを構築することができました。
次回はKubernetesに、[GuestBook example](https://github.com/GoogleCloudPlatform/kubernetes/blob/master/examples/guestbook/README.md)をデプロイしてみます。

