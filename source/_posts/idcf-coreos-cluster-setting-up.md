title: 'IDCFクラウドにCoreOSクラスタを構築する - Part3: セットアップ'
date: 2014-07-15 00:47:22
tags:
 - IDCFクラウド
 - CoreOS
 - fleet
 - fleetctl
description: 先月Dockerから、libswarmが発表されました。どちらかというとIaaSにインパクトがありそうです。永続化データの同期は課題として、バッチは海外の安いVPSを束ねて動かしてとか、リアルタイムや基幹システムは国内のハイスペックなインスタンスで動かしてとか、、the internet as one giant computerな世界はわくわくします。Orchestrationプラットフォームは、fleet, geard, Shipyard, Deimos, Kubernetes、Centurion、Heliosと百花繚乱の様相なのですが、ここはKubernetesとlibswarmでしょうか。
---


* `Update 2014-07-19`: [IDCFクラウドにCoreOSクラスタを構築する - Part5: 再セットアップ](/2014/07/19/idcf-coreos-cluster-re-setting-up/)



先月[libswarm](https://github.com/docker/libswarm)が発表されましたが、どちらかというとIaaSにインパクトがありそうです。

永続化データの同期は課題として、バッチは海外の安いVPSを束ねて動かしてとか、リアルタイムは国内のハイスペックなインスタンスで動かしてとか、、[the internet as one giant computer](http://www.wired.com/2013/09/docker/)な世界はわくわくします。

Orchestrationプラットフォームは、fleet, geard, Shipyard, Deimos, Kubernetes、Centurion、Heliosと百花繚乱の様相なのですが、ここはKubernetesとlibswarmでしょうか。

CoreOS、Kubernetes、Mesos、Saltをはやく組み合わせたいので、まずはCoreOSクラスタを動かせるようにします。

<!-- more -->

### cloud-config.ymlの準備

基本的には[IDCFクラウドでCoreOSをディスクからインストールする](/2014/06/03/idcf-coreos-install-disk/)と同じ手順ですが、CoreOSクラスタ管理のためfleet.serviceのunitを追加します。

CoreOSのSSHは公開鍵認証なのでを忘れずに書きます。この秘密鍵はssh-agentに追加してCoreOSクラスタ全体で使います。

``` yml ~/coreos_apps/cloud-config.yml
#cloud-config

ssh_authorized_keys:
  - ssh-rsa  AAAAB3NzaC1yc2EAAAADAQABA...

coreos:
  etcd:
      discovery: https://discovery.etcd.io/1461ea8c611984341b01f611ff5940e1
      addr: $private_ipv4:4001
      peer-addr: $private_ipv4:7001
  units:
    - name: etcd.service
      command: start
    - name: fleet.service
      command: start
```

### インスタンスを3台デプロイする

nameとdisplaynameを変更して、3台のCoreOSインスタンスを作成します。
とても面倒ですが、心折れずにテンプレートを用意します。

1. ディスクの作成
2. ディスクを作業用インスタンスにアタッチ
3. CoreOSをディスクにインストール
4. ディスクをデタッチ
5. ディスクからスナップショット作成
6. スナップショットからテンプレート作成
7. テンプレートからインスタンスの作成

``` bash
$ idcf-compute-api deployVirtualMachine \
  --serviceofferingid=21 \
  --templateid=8862 \
  --zoneid=1 \
  --name=coreos-cluster-v7-1 \
  --displayname=coreos-cluster-v7-1 \
  --group=mshimizu | jq '.deployvirtualmachineresponse.jobid'
```

IDCFクラウド上に3台のインスタンスができました。ここまで長かったですがようやくスタートラインです。

CoreOSクラスタを構築してしまえば、あとはetcdctlやfleetctlコマンドを使った操作になるので、どこのIaaSで動いているかは気にしなくてよくなります。


### fleetctlのセットアップ

作業マシンにfleetctlをセットアップします。
今回はDockerホストにインストールしますが、CoreOSクラスタの外側からfleetctlを使ってサービス(Dockerコンテナ)をデプロイすることができます。

``` bash
$ git clone https://github.com/coreos/fleet.git
$ cd fleet
$ ./build
$ sudo mv bin/fleetctl /usr/local/bin/
```

### fleetctlのオプション

`--endpoint`オプションにCoreOSノードの1台を指定して、etcdクラスタに直接接続します。

``` bash
$ fleetctl --endpoint 'http://10.1.1.45:4001'  list-machines
MACHINE         IP              METADATA
3a704dc1...     10.1.0.39       -
6e8e459f...     10.1.1.45       -
96704b20...     10.1.0.17       -
```

`--tunnel`オプションにCoreOSノードの1台を指定して、SSH経由でetcdクラスタに接続します。

``` bash
$ eval `ssh-agent`
$ ssh-key  ~/.ssh/my_key
$ fleetctl --tunnel 10.1.1.45 list-machines
MACHINE         IP              METADATA
3a704dc1...     10.1.0.39       -
6e8e459f...     10.1.1.45       -
96704b20...     10.1.0.17       -
```

CoreOSノードを経由して、別のCoreOSノードにSSH接続もできます。

``` bash
$ eval `ssh-agent`
$ ssh-key  ~/.ssh/my_key
$ fleetctl --tunnel 10.1.1.45 ssh 3a704dc1
CoreOS (beta)
```

### まとめ

今回のCoreOSクラスタ上にDeisを作りながら、fleetの使い方を勉強していきたいと思います。



