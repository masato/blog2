title: 'Vagrant on IDCFクラウド (32bit VirtualBox)'
date: 2014-05-10 06:02:09
tags:
 - IDCFクラウド
 - Vagrant
 - VirtualBox
description: 前回は、64bitのゲストOSにVagrantとLXCをインストールしました。今回もIDCFクラウドを使って、32bitのゲストOSに、VirtualBoxをインストールしてみます。
---

[前回](/2014/05/09/vagrant-on-idcf-64bit-lxc)は、64bitのゲストOSにVagrantとLXCをインストールしました。
今回も[IDCFクラウド](http://www.idcf.jp/cloud/service/self.html)を使って、32bitのゲストOSに、VirtualBoxをインストールしてみます。
<!-- more -->
IDCFクラウドでは、32bitのUbuntuテンプレートが提供されていないため、[ISO](http://www.idcf.jp/cloud/support/self/manual/05.html)をアップロードしてからインスタンスを作成します。
ubuntu-[PC (Intel x86) server install CD](http://releases.ubuntu.com/precise/ubuntu-12.04.4-server-i386.iso)の、URLを指定します。

### VirtualBoxのインストール

VirtualBoxと、DKMSでビルドするカーネルモジュールのソースをインストールします。

``` bash
$ sudo apt-get update
$ sudo apt-get install dpkg-dev virtualbox-dkms
```
VirtualBoxのバージョン確認。

``` bash
$ VBoxManage --version
4.1.12_Ubuntur77245
```

### Vagrantのインストール

32bit版のVagrantをインストールします。

``` bash
$ wget https://dl.bintray.com/mitchellh/vagrant/vagrant_1.6.1_i686.deb
$ sudo dpkg -i vagrant_1.6.1_i686.deb
```
VirtualBoxのカーネルモジュールをビルドします。

``` bash
$ sudo apt-get install linux-headers-$(uname -r)
$ sudo dpkg-reconfigure virtualbox-dkms
```

boxを追加します。

``` bash
$ vagrant box add precise32 http://files.vagrantup.com/precise32.box
```

### Vagrantfile

Vagrantfileを作成します。

``` bash
$ mkdir ~/test_project
$ cd !$
$ vagrant init
```

precise32をBoxに指定します。

``` ruby Vagrantfile
#config.vm.box = "base"
config.vm.box = "precise32"
```

### VMの起動

VMを起動して、SSHで接続します。

``` bash
$ vagrant up
$ vagrant ssh
Welcome to Ubuntu 12.04 LTS (GNU/Linux 3.2.0-23-generic-pae i686)
```

### VMの停止とBoxの削除

VMを停止して削除後に、Boxも削除します。

``` bash
$ vagrant halt
$ vagrant destroy
$ vagrant box remove precise32
```

### まとめ

32bitのVirtualBoxは起動に時間がかかるため、SSHの接続待ちがタイムアウトし、何回かリトライが必要でした。
VPS上にVagrantの開発環境が構築できました。Dockerのパッケージは64bit版しか提供されていないので、32bit版はビルドする必要があります。
[Docker on i386](http://mwhiteley.com/linux-containers/2013/08/31/docker-on-i386.html)に手順があったので、Goの勉強のためコードを読んでみるやも。


