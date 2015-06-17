title: 'IDCFクラウドでvagrant-lxcを使う'
date: 2014-05-11 19:35:08
tags:
 - IDCFクラウド
 - Vagrant
 - vagrant-lxc
description: 以前構築した、IDCFクラウド上のvagrant-lxcでCentOSを使ってみます。公式には、UbuntuとDebianのBoxしかありません。でも本番環境はCentOSばかりなので、開発環境もCentOSで揃えたいときがあります。
---
[以前](/2014/05/09/vagrant-on-idcf-64bit-lxc)構築した、IDCFクラウド上のvagrant-lxcでCentOSを使ってみます。
[公式](https://github.com/fgrehm/vagrant-lxc-base-boxes)には、UbuntuとDebianのBoxしかありません。でも本番環境はCentOSばかりなので、開発環境もCentOSで揃えたいときがあります。

<!-- more -->

vagrant-lxc 1.0+では、[centos-vagrant-lxc](https://github.com/BashtonLtd/centos-vagrant-lxc)が動作しなくなり困ったのですが、
[vagrant-lxc用のCentOS Boxを自作する方法](http://www.ryuzee.com/contents/blog/6796)で紹介されているBoxは起動できました。

### Boxの追加
CentOS6.4のBoxをダウンロードします。

``` bash
$ vagrant box add centos64 https://dl.dropboxusercontent.com/u/428597/vagrant_boxes/vagrant-lxc-CentOS-6.4-x86_64-ja.box
```

### Vagrantfileの作成
プロジェクトのディレクトリを作成します。

``` bash
$ mkdir -p ~/vagrant_apps/centos64
$ cd !$
```
Vagrantfileを作成します。

``` ruby ~/vagrant_apps/centos64/Vagrantfile
Vagrant.configure("2") do |config|
  config.vm.box = "centos64"
end
```

### VMの起動とSSH接続
providerオプションにlxcを指定します。deprecatedのWARNINGが出ますが、VMを起動できました。

``` bash
$ vagrant up --provider=lxc
Bringing machine 'default' up with 'lxc' provider...
==> default: Importing base box 'centos64'...
==> default: WARNING: You are using a base box that has a format that has been deprecated, please upgrade to a new one.
==> default: Setting up mount entries for shared folders...
    default: /vagrant => /home/matedev/vagrant_apps/centos64
==> default: Starting container...
==> default: Waiting for machine to boot. This may take a few minutes...
    default: SSH address: 10.0.3.126:22
    default: SSH username: vagrant
    default: SSH auth method: private key
==> default: Machine booted and ready!
```
SSHで接続します。

``` bash
$ vagrant ssh
[vagrant@CentOS-6 ~]$
```

### まとめ
vagrant-lxcのBoxはもともと少ないのですが、1.0+になって以前のBoxが使えなくなってしまいました。
Boxは自分で作れるように勉強しないと継続して使えないです。ここで[packer-builder-lxc](https://github.com/kelseyhightower/packer-builder-lxc)の出番かやも。
