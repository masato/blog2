title: "Salt on IDCF クラウド - Part6: Dockerインストール"
date: 2014-11-22 19:58:47
tags:
 - Salt
 - Ubuntu
 - Docker
 - IDCFクラウド
 - IDCFCloud
 - SaltFormula
description: Salt Masterの設定ファイルを変更してSalt Statesの配置ディレクトリを変更しました。これまで何回かSaltを使ってDockerをインストールしていますが、あらためて整理してみます。
---

Salt Masterの設定ファイルを変更してSalt Statesの配置ディレクトリを変更しました。これまで[何回か](/2014/09/06/salt-idcf-docker-states/)Saltを使ってDockerをインストールしていますが、あらためて整理してみます。

<!-- more -->

### Saltのバージョンの確認

masterのバージョンを確認します。

```
$ salt-master --version
salt-master 2014.7.0 (Helium)
```

minionのバージョンを確認します。

``` bash 
$ salt 'minion*' cmd.run 'salt-minion --version'
minion2.cs74idcfcloud.internal:
    salt-minion 2014.7.0 (Helium)
minion1.cs74idcfcloud.internal:
    salt-minion 2014.7.0 (Helium)
```

### master.conf

master.confを編集して、masterの設定ファイルのルートディレクトリを変更します。`users-formula`は[前回](/2014/11/21/salt-on-idcf-cloud-users-formula/)と同じように、`git clone`します。

``` bash /etc/salt/master.d/master.conf
pillar_roots:
  base:
    - /srv/salt/config/pillar

file_roots:
  base:
    - /srv/salt/config/salt
    - /srv/salt/config/formulas/users-formula
```

### Statesの設定ファイルを変更

top.slsにdockerのFormulaを追加します。

``` yml /srv/salt/config/salt/top.sls
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
    - docker
```

common.slsには、vimへのalternativesを設定します。どうもnanoは使いづらいです。

``` yml /srv/salt/config/salt/common.sls
ja_JP.UTF-8:
  locale.system

Asia/Tokyo:
  timezone.system

editor:
  alternatives.set:
    - path: /usr/bin/vim.basic
```

common.slsはdocker-pyのインストールに必要なpipを共通設定としてインストールします。

``` yml /srv/salt/config/salt/docker/common.sls
pkg-core:
  pkg.installed:
    - refresh: True
    - pkgs:
      - git
      - python-pip
      - python-dev
```

### Docker

DockerのFormulaは[以前](/2014/09/06/salt-idcf-docker-states/)から見直しました。

``` yml /srv/salt/config/salt/docker/init.sls
include:
  - common

docker-py:
  pip.installed:
    - require:
      - pkg: pkg-core

docker-repo:
  pkgrepo.managed:
    - humanname: Docker Repo
    - name: deb https://get.docker.io/ubuntu docker main
    - key_url: https://get.docker.io/gpg
    - require:
      - pkg: pkg-core

apt-transport-https:
  pkg.installed

lxc-docker:
  pkg.latest:
    - require:
      - pkgrepo: docker-repo
      - pkg: apt-transport-https

docker:
  service:
    - running
    - enable: True
    - watch:
      - pkg: lxc-docker

jpetazzo/nsenter:
  docker.pulled

nsenter:
  docker.installed:
    - image: jpetazzo/nsenter
    - volumes:
      - /usr/local/bin: /target

/usr/local/bin/nse:
  file.managed:
    - source: salt://docker/nse
    - mode: 775
```

Docker1.3になるとあまり出番がありませんが、nsenterのラッパースクリプトを用意します。

``` bash /srv/salt/config/salt/docker/nse
[ -n "$1" ] && sudo nsenter --pid --uts --mount --ipc --net --target $(docker inspect --format="&#123;&#123; .State.Pid }}" $1)
```
### higistate

state.highstateを実行して、`roles:dev`のminionに対してDockerをインストールします。

``` bash
$ salt '*' state.highstate
```

### 確認

cmdモジュールを使い、Grainsに定義した`roles:dev`のminionにインストールしたdockerのバージョンを確認します。

``` bash
$ salt -G 'roles:dev' cmd.run 'docker version'
minion2.cs74idcfcloud.internal:
    Client version: 1.3.1
    Client API version: 1.15
    Go version (client): go1.3.3
    Git commit (client): 4e9bbfa
    OS/Arch (client): linux/amd64
    Server version: 1.3.1
    Server API version: 1.15
    Go version (server): go1.3.3
    Git commit (server): 4e9bbfa
minion1.cs74idcfcloud.internal:
    Client version: 1.3.1
    Client API version: 1.15
    Go version (client): go1.3.3
    Git commit (client): 4e9bbfa
    OS/Arch (client): linux/amd64
    Server version: 1.3.1
    Server API version: 1.15
    Go version (server): go1.3.3
    Git commit (server): 4e9bbfa
```
