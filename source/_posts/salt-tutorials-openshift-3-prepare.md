title: 'Salt チュートリアル - Part3: OpenShift Origin v3の準備'
date: 2014-09-12 00:02:52
tags:
 - Ubuntu
 - Salt
 - Kubernetes
 - OpenShiftv3
 - OpenShiftOrigin
 - OpenShift
 - pip
description: KubernetesをベースにしたOpenShift Origin v3のリポジトリを読んでいると、OpenShift Origin V4とは別のGoで書かれたプログラムです。KubernetesがインストールできるLinuxならRHEL7やFedora20でなくても動きそうです。OpenShift Origin 3をインストールするUbuntu14.04サーバーを構築します。Prerequisitesによると、GoとDockerが事前に必要なため、Saltでminionの構成管理を行います。
---

Kubernetesをベースにした`OpenShift Origin v3`の[GitHubリポジトリ](https://github.com/openshift/origin)を読んでいると、`OpenShift Origin V4`とは別のGoで書かれたプログラムです。KubernetesがインストールできるLinuxならRHEL7やFedora20でなくても動きそうです。

`OpenShift Origin v3`をインストールするUbuntu14.04サーバーを構築します。
[Prerequisites](https://github.com/openshift/origin/blob/master/CONTRIBUTING.adoc)によると、GoとDockerが事前に必要なため、Saltでminionの構成管理を行います。

<!-- more -->

### Minion

`roles`をminionの静的な設定として使うため、grainsの設定をします。
minionにログインして設定ファイルを作成します。

``` yaml /etc/salt/minion.d/grains.conf
grains:
  roles:
    - openshift
```

### Masterでgrainsマッチ

masterのtop.slsに、minionのgrainsがmatchした場合に実行するstatesの指定を書きます。
minionに設定したrolesが`openshift`の場合、GoのSLSファイルを実行します。

``` yaml /srv/salt/top.sls
base:
  '*':
    - base
    - docker
  'roles:openshift':
    - match: grain
    - go
```

GoをインストールするSLSファイルです。
`refresh: True`を指定して常に最新のパッケージをインストールします。

``` yaml /srv/salt/go.sls
golang:
  pkg.latest:
    - refresh: True
```

### Ubuntu14.04にupgrade後のpipのエラー

minionのUbuntuを12.04から14.04に`do-release-upgrade`した後、pip.installedのstateがエラーになりました。
[pip stops with ImportError for request-Modul](https://bugs.launchpad.net/ubuntu/+source/python-pip/+bug/1306991)にissueがありますが、とりあえずWorkaroundで対応します。apt-getでremoveしてから、get-pipを使いインストールし直します。

``` bash
$ sudo apt-get remove python-pip
Removing python-pip (1.5.4-1) ...
Processing triggers for man-db (2.6.7.1-1) ...
$ wget https://raw.github.com/pypa/pip/master/contrib/get-pip.py
Downloading/unpacking pip
  Downloading pip-1.5.6-py2.py3-none-any.whl (1.0MB): 1.0MB downloaded
Installing collected packages: pip
Successfully installed pip
Cleaning up...
$ sudo python get-pip.py
```

### Docker States

[Docker States](/2014/09/06/salt-idcf-docker-states/)は前回と同じです。

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

### Masterのbase.sls

base.slsも少し修正しました。
`file`, `user`, `ssh_auth`のstatesを使い共通設定を行います。

* dockerグループの作成
* ユーザーの作成、dockerグループへの参加
* SSH公開鍵の設定
* GOPATHの設定

``` yaml /srv/salt/base.sls
group_docker:
  group.present:
    - name: docker
    - gid: 999
    - system: True
    - addusers:
      - mshimizu
    - require_in:
      - user: user_mshimizu

user_mshimizu:
  user.present:
    - shell: /bin/bash
    - name: mshimizu
    - fullname: Masato Shimizu
    - uid: 1000
    - home: /home/mshimizu
    - require_in:
      - ssh_auth: sshkey_mshimizu
      - file: /home/mshimizu/go

sshkey_mshimizu:
  ssh_auth:
    - present
    - user: mshimizu
    - enc: ssh-rsa
    - comment: mykey
    - name: AAAAB3NzaC1y...

/home/mshimizu/go:
  file.directory:
    - user: mshimizu
    - require_in:
      - file: /home/mshimizu/.profile

/home/mshimizu/.profile:
  file.append:
    - name: /home/mshimizu/.profile
    - text:
      - export GOPATH=$HOME/go
      - export PATH=$PATH:$GOPATH/bin

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

### Masterでhighstate実行

Masterでstate.highstateを実行します。
`-G`フラグでhighstateを実行するminionのgrainsを指定します。

``` bash
$ sudo salt -G 'roles:openshift' state.highstate
...
Summary
-------------
Succeeded: 16
Failed:     0
-------------
Total:     16
```