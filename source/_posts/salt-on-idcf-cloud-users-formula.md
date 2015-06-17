title: "Salt on IDCF クラウド - Part5: users-formulaを使う"
date: 2014-11-21 20:58:43
tags:
 - Salt
 - Ubuntu
 - IDCFクラウド
 - IDCFCloud
 - SaltFormula
description: 構成管理ツールとしての役割の一つにユーザー管理があります。複数のサーバーにユーザーを追加したり、公開鍵を設定する作業が自動化できると便利になります。あらかじめ定義されたSalt FormulaはGitHubのsaltstack-formulasリポジトリにあります。今回はusers-formulaを使いユーザー管理を行います。
---

構成管理ツールとしての役割の一つにユーザー管理があります。複数のサーバーにユーザーを追加したり、公開鍵を設定する作業が自動化できると便利になります。あらかじめ定義された`Salt Formula`はGitHubの[saltstack-formulas](https://github.com/saltstack-formulas)リポジトリにあります。今回は[users-formula](https://github.com/saltstack-formulas/users-formula)を使いユーザー管理を行います。

<!-- more -->

### GitFSからFormulaを使う場合

Salt FormulaをGitHubからインストールする場合、直接`git clone`を使う方法と、[GitFS](http://docs.saltstack.com/en/latest/topics/tutorials/gitfs.html#tutorial-gitfs)を使う方法があります。GitFSを使う場合は事前にpygit2をmasterのサーバーにインストールします。

### 直接 git clone してFormulaを使う場合

今回は簡単に、直接ファイルシステムにgit clone します。

### users-formulaをインストールする

ユーザー管理のために[users-formula](https://github.com/saltstack-formulas/users-formula)を使います。Formulaをダウンロードするディレクトリを作成して、Formulaをgit cloneします。

``` bash
$ mkdir -p /srv/formulas
$ cd !$
$ git clone https://github.com/saltstack-formulas/users-formula.git
```

git cloneしたディレクトリを、[file_roots](http://docs.saltstack.com/en/latest/ref/configuration/master.html#std:conf_master-file_roots)のbase追加します。

``` yml /etc/salt/master.d/master.conf
file_roots:
  base:
    - /srv/salt
    - /srv/formulas/users-formula
```

`/etc/salt/master.d/master.conf`を編集したので、salt-masterをリスタートします。

``` bash
$ service salt-master restart
```

### Pillarの記述

Pillarのtop.slsにusersを追加します。

``` yml /srv/pillar/top.sls
base:
  '*':
    - users
    - s3cmd
```

[users-formulaのpillar.example](https://github.com/saltstack-formulas/users-formula/blob/master/pillar.example)を参考にして、作成するユーザーのデータをPillarに記述します。

``` yml /srv/pillar/users/init.sls
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
      - ssh-rsa AAAAB3Nxxx
```

### users-formulaを実行する

users-formulaをテストします。

``` bash
# salt '*' state.show_sls users
```

Salt Formulaのtop.slsに、users-formulaを追加します。

``` yml /srv/salt/top.sls
base:
  '*':
    - base
    - users
  'roles:salt-master':
    - match: grain
  'roles:dev':
    - match: grain
    - emacs
    - s3cmd
```

highstateを実行してFormulaを適用します。

```
$ salt '*' state.highstate
```
