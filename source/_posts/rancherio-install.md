title: "Rancher.ioでDockerコンテナを管理する - Part3: Rancher.ioをインストールする"
date: 2015-01-07 01:29:55
tags:
 - Docker管理
 - Rancherio
 - Cattle
 - Ubuntu
 - Salt
 - IDCFクラウド
description: IDCFクラウド上に構築したSalt+Docker環境にRancher.ioをデプロイします。Rancher.ioは管理サービスもDockerコンテナとしてデプロイするので、インストールがとても簡単です。salt-masterにManagement Serverを、salt-minionにDocker Nodesをデプロイします。
---

IDCFクラウド上に構築した[Salt+Docker](/2015/01/06/rancherio-install-salt-docker/)環境に[Rancher.io](https://github.com/rancherio/rancher)をデプロイします。Rancher.ioは管理サービスもDockerコンテナとしてデプロイするので、インストールがとても簡単です。salt-masterにManagement Serverを、salt-minionにDocker Nodesをデプロイします。

<!-- more -->


### Saltクラスタの確認

minionの仮想マシンは以下の2台です。salt.cs3d0idcfcloud.internalの仮想マシンはsalt-masterとsalt-minionを兼用しています。

``` bash
$ salt-key
Accepted Keys:
minion1.cs3d0idcfcloud.internal
salt.cs3d0idcfcloud.internal
Unaccepted Keys:
Rejected Keys:
root@salt:/srv/salt/config/salt#
```

### Management Serverのインストール

salt-masterの仮想マシン(salt.cs3d0idcfcloud.internal)にログインします。Dockerイメージをpullして起動するだけです。

``` bash
$ sudo docker pull rancher/server
$ sudo docker run -d -p 8080:8080 rancher/server
```

### Docker Nodesのインストール

salt-minionの仮想マシン(minion1.cs3d0idcfcloud.internal)にログインします。こちらもDockerイメージをpullして起動するだけですが、いくつか引数を渡します。

``` bash
$ sudo docker pull rancher/agent
$ sudo docker run  -it \
  -e CATTLE_AGENT_IP="210.140.173.117" \
  -v /var/run/docker.sock:/var/run/docker.sock rancher/agent \
  http://10.3.0.165:8080
```

### CATTLE_AGENT_IP

まず、環境変数の`CATTLE_AGENT_IP`は`Docker Nodes`の仮想マシンにスタティックNATしたパブリックIPアドレスを指定します。IDCFクラウドの場合アカウントのVLANとインターネットの間にVirtual Routerがあります。仮想マシンのプライベートIPアドレスは直接インターネットの外部から接続できません。あらかじめポータル画面からパブリックIPアドレスの1つをminion1の仮想マシンに対してスタティックNATしておく必要があります。`CATTLE_AGENT_IP`にはこのスタティックNATを指定して、外部から接続できるIPアドレスを割り当てます。このIPアドレスはRancher.ioのUI(管理画面)に`Host`のIPアドレスに表示されます。ブラウザからWebSocketを使ったコンソール接続や、cAdvisorを使ったメトリクスの収集に直接使用されます。

### Management ServerのIPアドレス

上記では`http://10.3.0.165:8080`が`Management Server`のIPアドレスになります。`Docker Nodes`からの接続になるためアカウントVLAN内のプライベートIPアドレスで指定します。

### UI (Rancher.ioの管理画面)

`Docker Nodes`のコンテナをsalt-minionで起動してしばらくするとUIにHostとして表示されます。

{% img center /2015/01/07/rancherio-install/rancher-ui-hosts-top.png %}

### Ghostイメージのデプロイ

サンプルとして[dockerfile/ghost/](https://registry.hub.docker.com/u/dockerfile/ghost/)を使います。通常の`docker run`コマンドの場合は以下に相当します。

``` bash
$ docker run -d -p 80:2368 dockerfile/ghost
```

### Create Containerダイアログ

Hostの`No container yet.`の下のプラスボタンを押して`Create Container`ダイアログを表示します。

The BasicタブにDocker Hub Registryから取得するイメージ情報など基本情報を入力します。

{% img center /2015/01/07/rancherio-install/rancher-container-basic.png %}

Portタブに、Dockerホストにマップするポートを入力します。

{% img center /2015/01/07/rancherio-install/rancher-container-port.png %}

Commandタブの`As User`は必須項目になっています。ブランクだとコンテナが作成できないため参考値の`root`を指定します。

{% img center /2015/01/07/rancherio-install/rancher-container-command.png %}

### cAdvisorのリアルタイムグラフ

cAdvisorのメトリクスをブラウザからWebSocketでCattle Agent経由で取得します。`Docker Nodes`のホストと、コンテナのメトリクスがそれぞれリアルタイムで表示されます。

Hostのグラフです。

{% img center /2015/01/07/rancherio-install/rancher-ui-hosts.png %}

コンテナのグラフです。

{% img center /2015/01/07/rancherio-install/rancher-container-ghost.png %}

### コンテナのシェル操作

各Containerページの右上、`Execute Shell`アイコンをクリックするとWebSocket経由でコンテナのシェルを実行することができます。

{% img center /2015/01/07/rancherio-install/rancher-container-execute-shell.png %}

コンテナのコンソールはlibvirtのVNC WebSocketサポートを使っているようです。

{% img center /2015/01/07/rancherio-install/rancher-container-shell.png %}