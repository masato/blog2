title: "CoreOSをfleetctlから操作する - Part1: --endpointと--tunnellオプション"
date: 2014-07-25 22:58:06
tags:
 - CoreOS
 - fleet
 - fleetctl
description: IDCFクラウドにCoreOSクラスタを構築する - Part5 再セットアップでようやく構築できたCoreOSクラスタを使ってみます。最初にCoreOSを操作するために、fleetctlをCoreOSクラスタ外部にインストールして使います。
---

[IDCFクラウドにCoreOSクラスタを構築する - Part5: 再セットアップ](/2014/07/19/idcf-coreos-cluster-re-setting-up/)でようやく構築できたCoreOSクラスタを使ってみます。
最初にCoreOSを操作するために、fleetctlをCoreOSクラスタ外部にインストールします。

<!-- more -->

### fleetctlのインストール

CoreOSクラスタ外部のワークステーションにfleetctlをインストールします。

``` bash
$ git clone https://github.com/coreos/fleet.git
$ cd fleet
$ ./build
$ sudo mv bin/fleetctl /usr/local/bin/
```

### --endpointオプション

`--endpoint`を指定して、直接CoreOSクラスタに接続します。
URLはCoreOSクラスタの1台のetcdを指定します。

``` bash
$ fleetctl --endpoint http://10.1.0.174:4001 list-machines
MACHINE         IP              METADATA
0ba88b13...     10.1.0.174      -
3b9bf346...     10.1.1.90       -
5d2a8312...     10.1.0.117      -
```

### --tunnellオプション

`--tunnell`を指定して、SSH経由で外部から接続します。

IPアドレスはCoreOSクラスタの1台を指定します。
ssh-agentを使うのでCoreOSのSSH接続に使う秘密鍵を追加してから接続します。

``` bash
$ eval `ssh-agent`
$ ssh-add  ~/.ssh/my_key
$ fleetctl --tunnel 10.1.0.174 list-machines
MACHINE         IP              METADATA
0ba88b13...     10.1.0.174      -
3b9bf346...     10.1.1.90       -
5d2a8312...     10.1.0.117      -
```

### fleetctl ssh

CONTAINER_IDを指定してDockerコンテナにSSH接続します。

``` bash
$ eval `ssh-agent`
$ ssh-add  ~/.ssh/my_key
$ fleetctl --tunnel 10.1.0.174 ssh 0ba88b13
CoreOS (beta)
core@coreos-beta-v3-1 ~ $
```

### FLEETCTL_ENDPOINT と FLEETCTL_TUNNEL

環境変数FLEETCTL_ENDPOINTまたは、FLEETCTL_TUNNELを設定すると、オプションを省略できます。

``` bash
$ export FLEETCTL_ENDPOINT=http://10.1.0.174:4001
$ fleetctl list-machines
MACHINE         IP              METADATA
0ba88b13...     10.1.0.174      -
3b9bf346...     10.1.1.90       -
5d2a8312...     10.1.0.117      -
```

またはFLEETCTL_TUNNELをexportします。

``` bash
$ export FLEETCTL_TUNNEL=10.1.0.174
$ fleetctl list-machines
MACHINE         IP              METADATA
0ba88b13...     10.1.0.174      -
3b9bf346...     10.1.1.90       -
5d2a8312...     10.1.0.117      -
```
