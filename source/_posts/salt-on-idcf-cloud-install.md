title: 'Salt on IDCF クラウド - Part1: インストールとGrain'
date: 2014-11-17 22:49:51
tags:
 - Salt
 - IDCFクラウド
 - IDCFCloud
 - ZeroMQ
 - Ubuntu
 - SaltGrain
description: プロダクション環境でCoreOSが使えない場合や、ホスト1台で構わない場合は、Saltで構成管理をしてsalt-minionにDockerホストをインストールして使っています。今回はVPSでなくIDCFクラウドを使います。FWの設定をしなくてもVLAN内で安全な通信と名前解決ができるので、Saltクラスタを作る場合に便利です。
---

プロダクション環境でCoreOSが使えない場合や、ホスト1台で構わない場合は、Saltで構成管理をしてsalt-minionにDockerホストをインストールして使っています。今回はVPSでなくIDCFクラウドを使います。FWの設定をしなくてもVLAN内で安全な通信と名前解決ができるので、Saltクラスタを作る場合に便利です。

<!-- more -->

### Saltについて

SaltはPythonで書かれた構成管理ツールです。CoreOSを使っているとdisposableなホストの使い方に慣れてきて、OSインストール後にホストの構成管理をすることは少なくなっていくと思いますが、従来型の構成管理が必要な場合にSaltは便利です。

日本ではなぜかPuppetよりもChefの方が人気ですが、Linux foundationが発表したオープンソースプロジェクトの[2014年ランキング](https://www.linux.com/news/enterprise/cloud-computing/784573-the-top-open-source-cloud-projects-of-2014)をみると、AnsibleやSaltの人気も高まっています。1位がPuppetの23%、2位がAnsibleの18%に続いて、Saltは投票率13.3%で3位です。また、Kubernetesの管理サービスのインストールに使われているので知名度も上がっています。

Saltの特徴です。

* YAML形式でサーバーの状態を設定する
* masterとminionで構成される
* ZeroMQで暗号化通信するするため、SSHがなくても通信できる
* インストールとクラスタ構築が簡単
* ReactorやEventを使ったイベント駆動システム
* 組み込みのモジュールが豊富に用意されている

### ZeroMQ

特にZeroMQが便利です。通信はAESで暗号化されています。SSHを使ないため`sshd_cofig`を間違えてsshdが起動しなくなった場合でもminionと通信して、sshdを起動することができます。

``` bash
$ salt 'minion1*' cmd.run "service ssh stop"
minion1.cs74idcfcloud.internal:
    ssh stop/waiting
$ salt 'minion1*' cmd.run "service ssh status"
minion1.cs74idcfcloud.internal:
    ssh stop/waiting
$ salt 'minion1*' cmd.run "service ssh start"
minion1.cs74idcfcloud.internal:
    ssh start/running, process 11068
```

### Ubuntu14.04のインスタンスを3台用意

Ubuntu 14.04のインスタンスを3台用意します。masterのノードにはminionもインストールしました。masterはhostnameを`salt`にすると、minionが自動的にmasterを見つけて公開鍵の登録リクエストします。

| Role          | hostname  | IP         |
|:-------------:|:---------:|-----------:|
| master,minion | salt      | 10.3.0.108 |
| minion        | minion1   | 10.3.0.97  |
| minion        | minion2   | 10.3.0.60  |

### Salt Bootstrap

[Salt Bootstrap](http://docs.saltstack.com/en/latest/topics/tutorials/salt_bootstrap.html)はワンライナーのインストールスクリプトです。フラグを追加してインストールのオプション設定ができます。

* -X: インストール後にデーモンを起動しない
* -M: masterもインストールする
* -N: minionをインストールしない
  
### masterのインストール

以下の作業はすべてrootで行います。まず`-M`フラグをつけてmasterをインストールします。

``` bash
$ curl -L http://bootstrap.saltstack.com | sh -s -- -M
```

### minionのインストール

今回はmasterのhostnameを`salt`にしているので自動的に見つけますが、通常はminionの設定ファイルにmasterの名前を登録しておきます。

``` bash
$ mkdir -p /etc/salt/minion.d/
$ sh -c 'echo master: salt > /etc/salt/minion.d/master.conf'
```

minionをインストールします。`-X`フラグをつけてインストール後にすぐsalt-minionを起動させないようにしました。

``` bash
$ curl -L https://bootstrap.saltstack.com | sh -s -- -X
```

### minionにGrainsの登録

State System, Salt Formula, Pillars, Grains, Salt Syndic, Salt Roaster, Salt Reactorなど日本人には名前だけでは想像が付かない仕組みがたくさんあり、最初は戸惑います。必要な機能だけ使いながら少しずつ理解していくことにします。

今回はGrainsを使いminionのロールを定義してみます。Grainsはminionに登録する静的なデータのことで、いろいろな用途に使えます。


minion1とminion2ノードのminionには、`roles:dev`を定義します。

``` yml /etc/salt/minion.d/grains.conf
grains:
  roles:
    - dev
```

saltノードのminionには、`roles:salt-master`を定義します。

``` yml /etc/salt/minion.d/grains.conf
grains:
  roles:
    - salt-master
```

Grainsの設定が終了したら、salt-minionのサービスをstartします。

``` bash
$ service salt-minion start
```

### salt-minionの公開鍵を承認する

masterにログインして、minioの公開鍵を承認してSaltクラスタを構築します。`salt-key`コマンドは、`-L`フラグをつけると、全てのminionの公開鍵を表示します。最初の状態では承認された公開鍵はひとつもありませんが、すでにminionから公開鍵の登録リクエストが来ています。

``` bash
$ salt-key -L
Accepted Keys:
Unaccepted Keys:
minion1.cs74idcfcloud.internal
minion2.cs74idcfcloud.internal
salt.cs74idcfcloud.internal
Rejected Keys:
```

`-A`フラグを使うと、未承認のminionの公開鍵をすべて承認できます。

``` bash
$ salt-key -A
The following keys are going to be accepted:
Unaccepted Keys:
minion1.cs74idcfcloud.internal
minion2.cs74idcfcloud.internal
salt.cs74idcfcloud.internal
Proceed? [n/Y] Y
Key for minion minion1.cs74idcfcloud.internal accepted.
Key for minion minion2.cs74idcfcloud.internal accepted.
Key for minion salt.cs74idcfcloud.internal accepted.
```

最後に`-L`フラグでminionが承認されたことを確認します。

``` bash
$ salt-key -L
Accepted Keys:
minion1.cs74idcfcloud.internal
minion2.cs74idcfcloud.internal
salt.cs74idcfcloud.internal
Unaccepted Keys:
Rejected Keys:
```

### 組み込みモジュールのテスト

さっそくmasterから、すべてのminionでechoコマンドを実行するテストをしてみます。'*'はワイルドカードです。

``` bash
$ salt '*' cmd.run 'echo hello world'
minion2.cs74idcfcloud.internal:
    hello world
salt.cs74idcfcloud.internal:
    hello world
minion1.cs74idcfcloud.internal:
    hello world
```

個別にコマンドを実行する場合は以下のようにワイルドカードの条件をしぼったり、FQDNを指定します。

``` bash
$ salt 'minion2*' cmd.run 'echo hello world'
minion2.cs74idcfcloud.internal:
    hello world
$ salt 'minion1.cs74idcfcloud.internal' cmd.run 'echo hello world'
minion1.cs74idcfcloud.internal:
    hello world
```

または、Grainsを使って特定のminionに対して実処理を行することもできます。

``` bash
$ salt -G 'roles:dev' test.ping
minion1.cs74idcfcloud.internal:
    True
minion2.cs74idcfcloud.internal:
    True
```

cmdの他にも、[組み込みのモジュール一覧](http://docs.saltstack.com/en/latest/ref/modules/all/index.html#all-salt-modules)にたくさんのモジュールが用意されています。

aptpkgモジュールを使い、 APT データベースを最新にしてみます。

``` bash
$ salt 'minion1*' pkg.refresh_db
minion1.cs74idcfcloud.internal:
    ----------
    http://jp.archive.ubuntu.com trusty InRelease:
        False
    http://jp.archive.ubuntu.com trusty Release:
        None
    http://jp.archive.ubuntu.com trusty Release.gpg:
        None
    http://jp.archive.ubuntu.com trusty-backports InRelease:
...
```
