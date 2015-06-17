title: 'Vagrant on IDCFクラウド (64bit LXC)'
date: 2014-05-09 00:57:13
tags:
 - IDCFクラウド
 - vagrant-lxc
 - Vagrant
 - LXC
description: vagrant-digitaloceanを使うと、VPSをVagrantのproviderにすることができます。providerではなく、VPS上にVagrantをインストールできないか試してみたくて、IDCFクラウドを使ってみました。
---
[vagrant-digitalocean](https://github.com/smdahlen/vagrant-digitalocean)を使うと、VPSをVagrantのproviderにすることができます。
providerではなく、VPS上にVagrantをインストールできないか試してみたくて、[IDCFクラウド](http://www.idcf.jp/cloud/service/self.html)を使ってみました。
<!-- more -->

### Vagrantのインストール

64bit版のUbuntuへ、最初に依存パッケージをインストールします。Saucyを用意しました。

``` bash
$ sudo apt-get update
$ sudo apt-get install lxc redir curl wget lxc-templates cgroup-lite
```
Vagrantをインストールします。2014-05-09現在では、1.6.1が最新です。

``` bash
$ wget https://dl.bintray.com/mitchellh/vagrant/vagrant_1.6.1_x86_64.deb
$ sudo dpkg -i vagrant_1.6.1_x86_64.deb
```

### vagrant-lxcのインストール

[Digital Ocean](https://www.digitalocean.com/)も同様ですが、64bitのゲストOSには、ホスト側のVT-xが無効なため、VirtualBoxがインストールできません。
providerに[vagrant-lxc](https://github.com/fgrehm/vagrant-lxc)を使います。ちなみに、32bitのゲストOSへは、起動が遅くなりますがVirtualBoxがインストールできます。
``` bash
$ vagrant plugin install vagrant-lxc --plugin-version 1.0.0.alpha.2
```

### VMの起動

プロジェクトのディレクトリを作成します。

``` bash
$ mkdir -p ~/vagrant_apps/precise64
$ cd !$
```

vagrant-lxcのBOXは、[vagrant-lxc-base-boxes](https://github.com/fgrehm/vagrant-lxc-base-boxes)で確認できます。

``` bash
$ vagrant init fgrehm/precise64-lxc
$ vagrant up --provider=lxc
```

SSHで接続します。

``` bash
$ vagrant ssh
Welcome to Ubuntu 12.04.4 LTS (GNU/Linux 3.11.0-12-generic x86_64)
```

### まとめ

VirtualBoxは起動が遅いので、providerにLXCを使うと、Vagrantでも軽量な仮想マシンを作成できます。
大きめなVPSのVMを一つ用意しておけば、VPS内でVagrantが実行できるので、リソースを有効に使うことができます。

[coreos-vagrant](https://github.com/coreos/coreos-vagrant)を使いたかったのですが、
providerがVirtualBoxなので、LXCが使えなかったり、そもそもデフォルトのCoreOSやDockerは64bitのため、32bitのVirtualBoxでは動作しません。
CoreOSやfleetの写経はVagrantが多いので、いまのところVPSでは起動できませんでした。


