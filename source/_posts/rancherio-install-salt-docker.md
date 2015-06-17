title: "Rancher.ioでDockerコンテナを管理する - Part2: IDCFクラウドにSaltとDockerをインストールする"
date: 2015-01-06 15:06:35
tags:
 - Docker管理
 - Rancherio
 - Cattle
 - Ubuntu
 - Salt
 - IDCFクラウド
description: 以前少し調べてたDockerコンテナ管理ツールのRancher.ioをIDCFクラウドに構築していこうと思います。はじめにSaltを使ってDockerをインストールします。以前Azureに構築したSaltクラスタと同様にsalt-configのテンプレートを使います。IDCFクラウド上にsalt-masterとsalt-minion用の仮想マシンを1台ずつ作成しておきます。
---

以前少し調べてた[Dockerコンテナ管理ツールのRancher.io](/2014/12/14/rancherio-docker-for-container-service/)をIDCFクラウドに構築していこうと思います。はじめにSaltを使ってDockerをインストールします。[Azure](/2014/12/18/microsoft-azure-salt-docker-install/)に構築したSaltクラスタと同様に[salt-config](https://github.com/masato/salt-config)のテンプレートを使います。IDCFクラウド上にsalt-masterとsalt-minion用の仮想マシンを1台ずつ作成しておきます。

<!-- more -->

### SLSファイル

このテンプレートに用意しているSLSファイルは以下になります。

``` bash
$ tree /srv/salt/config/salt/
/srv/salt/config/salt/
├── base.sls
├── common.sls
├── docker
│   └── init.sls
├── emacs
│   └── init.sls
├── misc.sls
├── s3cmd
│   ├── init.sls
│   └── s3cfg
└── top.sls
```

top.slsにGrainで定義したロールごとにインストールするパッケージを分けています。Dockerはどちらのロールでもインストールされます。

``` yml /srv/salt/config/salt/top.sls
base:
  '*':
    - base
    - users
  'roles:salt-master':
    - match: grain
    - docker
  'roles:dev':
    - match: grain
    - emacs
    - s3cmd
    - docker
```

まだIDCFオブジェクトストレージは使いませんが、Pillarを編集してキー情報を設定すれば使えるようにしています。

``` bash
$ cd /srv/salt/config/pillar
$ cp s3cmd.sls.sample s3cmd.sls
$ vi s3cmd.sls
s3cmd:
  access_key: xxx
  secret_key: xxx
```


### salt-masterのインストール

salt-masterの仮想マシンにログインしてrootで作業します。ホスト名は`salt.cs3d0idcfcloud.internal`です。

``` bash
$ apt-get update && apt-get install -y git
$ git config --global user.name "Masato Shimizu" 
$ git config --global user.email "ma6ato@gmail.com" 
$ git config --global push.default simple
```

[salt-configure](https://github.com/masato/salt-config)のテンプレートを`git clone`します。

``` bash
$ git clone --recursive git@github.com:masato/salt-config.git /srv/salt/config
$ cd /srv/salt/config
```

作業ユーザーを作成します。`ssh_auth`は公開鍵を設定します。

``` bash
$ cd /srv/salt/config/pillar/users
$ cp init.sls.sample init.sls
$ vi init.sls
users:
  mshimizu:
    fullname: Masato Shimizu
    password: xxx
    sudouser: True
    sudo_rules:
      - ALL=(ALL) NOPASSWD:ALL
    groups:
      - docker
    ssh_auth:
      - ssh-rsa AAAAB3Nzxxx
```

SLSや設定ファイルをコピーします。Salt Formulaは[users-formula](https://github.com/saltstack-formulas/users-formula)を使うため、GitHubのリポジトリをサブモジュールにしています。

``` bash
$ mkdir -p /etc/salt/master.d
$ cp /srv/salt/config/etc/salt/master.d/master.conf /etc/salt/master.d
$ cat /etc/salt/master.d/master.conf
pillar_roots:
  base:
    - /srv/salt/config/pillar

file_roots:
  base:
    - /srv/salt/config/salt
    - /srv/salt/config/formulas/users-formula
```

salt-masterをインストールする仮想マシンにはsalt-minioもインストールします。Grainにsalt-minionのroleを定義します。

``` bash
$ mkdir -p /etc/salt/minion.d
$ cat <<EOF >  /etc/salt/minion.d/grains.conf
grains:
  roles:
    - salt-master
EOF
```

salt-masterをbootstrapからインストールします。salt-minionサービスも起動します。

``` bash
$ curl -L http://bootstrap.saltstack.com | sh -s -- -M
$ service salt-minion start
```

Saltは2014.7.0 (Helium)がインストールされました。

``` bash
$ salt --version
salt 2014.7.0 (Helium)
```

### salt-minionのインストール

salt-minionの仮想マシンにログインしてrootで作業します。Grainにsalt-minionのroleを定義します。ホスト名は`minion1.cs3d0idcfcloud.internal`です。

``` bash
$ mkdir -p /etc/salt/minion.d
$ cat <<EOF >  /etc/salt/minion.d/grains.conf
grains:
  roles:
    - dev
EOF
```

salt-minionをbootstrapからインストールしてサービスを起動します。

``` bash
$ curl -L https://bootstrap.saltstack.com | sh
$ service salt-minion start
```

### salt-keyで公開鍵の認証

salt-masterの仮想マシンにログインします。salt-keyコマンドを使い接続要求を出しているsalt-minionの公開鍵を承認します。

``` bash
$ salt-key -A
The following keys are going to be accepted:
Unaccepted Keys:
minion1.cs3d0idcfcloud.internal
salt.cs3d0idcfcloud.internal
Proceed? [n/Y] Y
Key for minion minion1.cs3d0idcfcloud.internal accepted.
Key for minion salt.cs3d0idcfcloud.internal accepted.
```

### state.highstate

state.highstateを実行して、すべてのSLSをsalt-minionに適応します。

``` bash
$ salt '*' state.highstate
```

### 確認

SaltでインストールしたDockerのバージョンを確認します。Dockerは1.4.1、Goは1.3.3がインストールされました。

``` bash
$ salt '*' cmd.run 'docker version'
salt.cs3d0idcfcloud.internal:
    Client version: 1.4.1
    Client API version: 1.16
    Go version (client): go1.3.3
    Git commit (client): 5bc2ff8
    OS/Arch (client): linux/amd64
    Server version: 1.4.1
    Server API version: 1.16
    Go version (server): go1.3.3
    Git commit (server): 5bc2ff8
minion1.cs3d0idcfcloud.internal:
    Client version: 1.4.1
    Client API version: 1.16
    Go version (client): go1.3.3
    Git commit (client): 5bc2ff8
    OS/Arch (client): linux/amd64
    Server version: 1.4.1
    Server API version: 1.16
    Go version (server): go1.3.3
    Git commit (server): 5bc2ff8
```


