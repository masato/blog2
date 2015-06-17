title: 'Salt チュートリアル - Part2: Reactor System'
date: 2014-09-10 00:04:14
tags:
 - Salt
 - Reactor
 - MessageBus
 - Kubernetes
description: Saltは構成管理ツールの側面もありますが、本来は分散環境で安全にPub-Subを行いリモートのコマンドを実行するためのツールです。そのためReactorやEventを使ったイベント駆動システムの側面もあります。Message Busが必要なシステムのインフラとしてSaltを使うのおもしろいです。
---

* `Update 2014-09-12`: [Salt チュートリアル - Part3: OpenShift Origin 3の準備](/2014/09/12/salt-tutorials-openshift-3-prepare/)

Saltは構成管理ツールの側面もありますが、本来はPUB/SUBで分散環境でリモートのコマンドを実行するためのツールです。
そのためReactorやEventを使ったイベント駆動システムの側面もあります。`Message Bus`が必要なシステムのインフラとしてSaltを使うのおもしろいです。

<!-- more -->

#### Reactor System

Saltはスケールする[ZeroMQを使ったPUB/SUB](http://zeromq.org/docs:labs#toc16)の良い実装例でもあります。
イベントハンドラもPythonで記述できるので、任意の処理をsalt-masterで実行できます。

### Kubernetesのreactor

ドキュメントを読んでもタグの`'salt/minion/*/start'`のマッチの意味を理解できなかったのですが、
[Infrastructure Management with SaltStack: Part 3 – Reactor and Events](http://vbyron.com/blog/infrastructure-management-saltstack-part-3-reactor-events/)に説明がありました。

> This tag says “every time any minion service starts”.

このタグで、minionが起動したときのイベントをmasterで受信することができます。

``` bash kubernetes/cluster/templates/salt-master.sh
cat <<EOF >/etc/salt/master.d/reactor.conf
# React to new minions starting by running highstate on them.
reactor:
  - 'salt/minion/*/start':
    - /srv/reactor/start.sls
EOF
```

イベントを受信すると、masterはイベントを発行したminionに対してhighstateを実行します。
Kubernetesの場合は、minion用のコンポーネントがインストールされます。

``` html kubernetes/cluster/saltbase/reactor/start.sls
# This runs highstate on the target node
highstate_run:
  cmd.state.highstate:
    - tgt: &#123;&#123; data['id'] }}
```
