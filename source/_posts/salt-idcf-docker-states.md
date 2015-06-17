title: 'Salt with Docker - Part1: Dockerインストール'
date: 2014-09-06 00:31:27
tags:
 - IDCFクラウド
 - Salt
 - Docker
description: IDCFクラウドのCLIでSaltをプロビジョニングするでsalt-masterとsalt-minionをインストールした環境をしばらく放置していました。Kubernetesの構築ではSaltが重要な位置を占めているのでSaltの復習をしたいと思います。ちょうど開発用のWordPressのコンテナを用意する必要があったので、Docker Statesを作成してminionにインストールします。
---

* `Update 2014-09-07`: [Salt with Docker - Part2: WordPressインストール](/2014/09/07/salt-idcf-docker-wordpress/)

[IDCFクラウドのCLIでSaltをプロビジョニングする](/2014/05/29/idcf-api-salt/)でsalt-masterとsalt-minionをインストールした環境をしばらく放置していました。Kubernetesの構築ではSaltが重要な位置を占めているのでSaltの復習をしたいと思います。
ちょうど開発用のWordPressのコンテナを用意する必要があったので、`Docker States`を作成してminionにインストールします。

<!-- more -->

### salt-minionのインスタンス作成

salt-minion用のインスタンスを作成してからインストールコマンドを実行します。
名前は`minion-wp`としました。

``` bash
$ salt-install minion-wp minion
...
salt-minion start/running, process 3179
Setting up debconf-utils (1.5.42ubuntu1) ...
 *  INFO: Running install_ubuntu_check_services()
 *  INFO: Running install_ubuntu_restart_daemons()
 *  INFO: Salt installed!
```

### salt-minion公開鍵をsalt-masterに登録

salt-masterのホストで、salt-minionの未承認の公開鍵を一覧します。
 
``` bash
$ sudo salt-key -L
Accepted Keys:
minion1.cs29dcloud.internal
minion2.cs29dcloud.internal
salt.cs29dcloud.internal
Unaccepted Keys:
minion-wp.cs29dcloud.internal
Rejected Keys:
```

作成したminion-wpの公開鍵を承認します。

``` bash
$ sudo salt-key -a minion-wp.cs29dcloud.internal
The following keys are going to be accepted:
Unaccepted Keys:
minion-wp.cs29dcloud.internal
Proceed? [n/Y] Y
Key for minion minion-wp.cs29dcloud.internal accepted.
```

再度一覧すると、すべて`Accepted Keys`に追加されています。

``` bash
$ sudo salt-key -L
Accepted Keys:
minion-wp.cs29dcloud.internal
minion1.cs29dcloud.internal
minion2.cs29dcloud.internal
salt.cs29dcloud.internal
Unaccepted Keys:
Rejected Keys:
```

### docker/init.sls

[Automating application deployments across clouds with Salt and Docker](http://thomason.io/automating-application-deployments-across-clouds-with-salt-and-docker/)を参考にして、`Salt States`を作成します。
[docker-formula](https://github.com/saltstack-formulas/docker-formula)もありますが、[salt.states.dockerio](http://docs.saltstack.com/en/latest/ref/states/all/salt.states.dockerio.html)を実行するための[docker-py](https://github.com/docker/docker-py)がインストールされないので、自分で定義することにします。

``` yaml /srv/salt/docker/init.sls
docker-python-pip:
  pkg.installed:
    - name: python-pip

docker-python-dockerpy:
  pip.installed:
    - name: docker-py
    - repo: git+https://github.com/dotcloud/docker-py.git
    - require:
      - pkg: pkg-core
      - pkg: docker-python-pip

docker-dependencies:
  pkg.installed:
    - pkgs:
      - iptables
      - ca-certificates
      - lxc

docker-repo:
  pkgrepo.managed:
    - humanname: Docker Repo
    - name: deb https://get.docker.io/ubuntu docker main
    - key_url: https://get.docker.io/gpg
    - require:
      - pkg: pkg-core

lxc-docker:
  pkg.latest:
    - require:
      - pkg: docker-dependencies

docker:
  service.running
```

### base.sls

共通のbase.slsは、[Kubernetes](https://github.com/GoogleCloudPlatform/kubernetes/blob/master/cluster/saltbase/salt/base.sls
)を参考にします。

localeは`ja_JP.UTF-8`を指定します。

``` yaml /srv/salt/base.sls
pkg-core:
  pkg.installed:
    - names:
      - apt-transport-https
      - python-apt
      - git
      - language-pack-ja

ja_JP.UTF-8:
  locale.system
```

### top.sls

top.slsにはbase環境を1つだけ定義します。

``` yaml /srv/salt/top.sls
base:
  '*':
    - base
    - docker
```

### state.highstate

salt-masterから`state.highstate`を実行して、minion-wpにDockerをインストールします。

``` bash
$ sudo salt 'minion-wp*' state.highstate
...
Summary
------------
Succeeded: 9
Failed:    0
------------
Total:     9
```
