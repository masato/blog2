title: "Salt on IDCF クラウド - Part7: まとめ"
date: 2014-11-23 21:58:45
tags:
 - Salt
 - IDCFクラウド
 - IDCFCloud
description: これまでの連載のまとめです。Salt on IDCF クラウド - Part6 Dockerインストール Salt on IDCF クラウド - Part5 users-formulaを使う Salt on IDCF クラウド - Part4 psutilとpsモジュールで監視 Salt on IDCF クラウド - Part3 PillarとIDCFオブジェクトストレージ用のs3cmd Salt on IDCF クラウド - Part2 Salt Formula Salt on IDCF クラウド - Part1 インストールとGrain 作成した設定ファイルは、GItHubのsalt-configリポジトリにあります。
---

これまでの連載のまとめです。

* [Salt on IDCF クラウド - Part6: Dockerインストール](/2014/11/22/salt-on-idcf-cloud-docker/)
* [Salt on IDCF クラウド - Part5: users-formulaを使う](/2014/11/21/salt-on-idcf-cloud-users-formula/)
* [Salt on IDCF クラウド - Part4: psutilとpsモジュールで監視](/2014/11/20/salt-on-idcf-cloud-monitoring/)
* [Salt on IDCF クラウド - Part3: PillarとIDCFオブジェクトストレージ用のs3cmd](/2014/11/19/salt-on-idcf-cloud-pillar-s3cmd/)
* [Salt on IDCF クラウド - Part2: Salt Formula](/2014/11/18/salt-on-idcf-cloud-salt-formula/)
* [Salt on IDCF クラウド - Part1: インストールとGrain](/2014/11/17/salt-on-idcf-cloud-install/)

作成した設定ファイルは、GitHubの[salt-config](https://github.com/masato/salt-config)リポジトリにあります。

<!-- more -->

### salt-configの使い方

最初にIDCFクラウドでUbuntu14.04のインスタンスを3台用意します。作成するときにhostnameを指定できるので、それぞれ以下のhostnameをつけます。

* salt
* minion1
* minion2

### master

masterにログインして、git cloneします。

``` bash
$ apt-get update && apt-get install -y git
$ git clone --recursive https://github.com/masato/salt-config.git /srv/salt/config
$ cd /srv/salt/config
```

Pillarの編集をします。IDCFオブジェクトストレージのaccess_keyとsecret_keyを指定します。

``` bash
$ cd /srv/salt/config/pillar
$ cp s3cmd.sls.sample s3cmd.sls
$ vi s3cmd.sls
s3cmd:
  access_key: xxx
  secret_key: xxx
```

users/init.slsのユーザー名、パスワード、公開鍵を環境にあわせて編集します。

``` bas
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
      - ssh-rsa AAAABxxx
```

/etc/salt/master.d/master.confをコピーします。

``` bas
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


masterのノードの/etc/salt/minion.d/grains.confを作成します。

``` bash
$ mkdir -p /etc/salt/minion.d
$ cat <<EOF >  /etc/salt/minion.d/grains.conf
grains:
  roles:
    - salt-master
EOF
```

bootstrapを使い、masterとminioをインストールします。

``` bash
$ curl -L http://bootstrap.saltstack.com | sh -s -- -M
$ service salt-minion start
```

### minion

minion1とminion2にログインしてそれぞれ以下の設定をします。/etc/salt/minion.d/grains.confを編集してroles:devを指定します。

``` bash
$ mkdir -p /etc/salt/minion.d
$ cat <<EOF >  /etc/salt/minion.d/grains.conf
grains:
  roles:
    - dev
EOF
$ curl -L https://bootstrap.saltstack.com | sh
$ service salt-minion start
```

### salt-key

masterでminionの公開鍵を承認します。

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

### highstate

最後にhighstateを実行します。

``` bash
$ salt '*' state.highstate
```

