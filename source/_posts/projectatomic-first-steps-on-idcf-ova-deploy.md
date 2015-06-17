title: "Project Atomic First Steps - Part2: IDCFクラウドにデプロイ"
date: 2014-09-15 01:28:44
tags:
 - ProjectAtomic
 - Fedora20
 - IDCFクラウド
 - OVA
 - Minnty
description: Fedora20でビルドしたAtomic Hostイメージを、OVA形式にコンバートしてIDCFクラウドにデプロイしてみましたが、config-driveを認識してくれないのでOSの起動が成功しませんでした。ビルド済みのqcow2イメージの場合はconfig-driveが不要なのでダウンロードしてIDCFクラウドにデプロイしてみます。
---

Fedora20で[ビルド](/2014/09/14/projectatomic-first-steps-on-idcf-fedora20-build/)した`Atomic Host`イメージを、OVA形式にしてIDCFクラウドにデプロイしてみましたが、config-driveを認識してくれませんでした。

ビルド済みの[qcow2イメージ](http://rpm-ostree.cloud.fedoraproject.org/project-atomic/images/f20/qemu/20140609.qcow2.xz)の場合はconfig-driveが不要なのでダウンロードしてIDCFクラウドにデプロイしてみます。

<!-- more -->

### qcow2ファイルをダウンロード

作業用のFedora20にログインして、qcow2イメージをダウンロード && 解凍します。

``` bash
$ wget http://rpm-ostree.cloud.fedoraproject.org/project-atomic/images/f20/qemu/20140609.qcow2.xz
$ xz -df 20140609.qcow2.xz
```

### vmdkファイルにコンバート

`qemu-img`コマンドを使い、qcow2ファイルをvmdkファイルにコンバートします。

``` bash
$ qemu-img convert -f qcow2 20140609.qcow2 -O vmdk 20140609-atomic.vmdk
```

### VMware Playerコンソールで作業

vmdkファイルを使った仮想マシンの起動方法は[前回](/2014/09/14/projectatomic-first-steps-on-idcf-fedora20-build/)と同じです。
rootはデフォルトでパスワードなしのため、最初にパスワード設定をします。

``` bash
# passwd root
```

### 管理者ユーザーとSSH設定

`VMware Player`コンソールは使いづらいので、`ip a`でIPアドレスを確認してからPuttyでSSH接続します。

一時的にrootのパスワード認証を許可します。

``` bash /etc/ssh/sshd_config
PermitRootLogin yes
```

sshdのunitをreloadします。

``` bash
# systemctl reload sshd.service
```

PuttyからパスワードでSSH接続して、管理ユーザーと公開鍵認証の設定をします。

``` bash
# useradd -m -d /home/mshimizu -s /bin/bash mshimizu
# echo "mshimizu ALL=(ALL) NOPASSWD:ALL" >> /etc/sudoers
# mkdir /home/mshimizu/.ssh/
# chmod 700 /home/mshimizu/.ssh
# vi /home/mshimizu/.ssh/authorized_keys
# chmod 600 /home/mshimizu/.ssh/authorized_keys
# chown -R mshimizu:mshimizu /home/mshimizu
```

SSHでrootのログインとパスワード認証を拒否します。

``` bash /etc/ssh/sshd_config
PermitRootLogin no
PasswordAuthentication no
```

sshdのunitをreloadします。

``` bash
# systemctl reload sshd.service
```

別のセッションのPuttyから管理者ユーザーで接続できることを確認します。

### OVAに変換

Windows7にインストールした[Vagrant環境のMintty](/2014/06/11/packer-windows-vagrant-mingw-mintty/)を起動します。

`Virtual Machines`のフォルダから作業フォルダに、vmdkファイルとvmxファイルをコピーします。

``` bash
$ mkdir ~/atomic
$ cd "C:\Users\masato\Documents\Virtual Machines\AtomicHostOVA"
$ cp 20140609-atomic.vmdk ~/atomic
$ cp AtomicHostOVA.vmx ~/atomic
```

vmxファイルの`virtualHW.version`を、IDCFクラウドのESXi4で使えるように`7`に変更します。

``` bash /home/masato/atomic/AtomicHostOVA.vmx
virtualHW.version = "7"
```

ovftoolのインストールを確認します。

``` bash
$ which ovftool
c:\Program Files\VMware\VMware OVF Tool\ovftool.EXE
```

ovftoolを使い、vmdkファイルとvmxファイルからOVAイメージを作成します。

``` bash
$ ovftool AtomicHostOVA.vmx AtomicHostOVA.ova
Opening VMX source: AtomicHostOVA.vmx
Opening OVA target: AtomicHostOVA.ova
Writing OVA package: AtomicHostOVA.ova
Transfer Completed
Completed successfully
```

### VMware PlayerでOVAのテスト

AtomicHostOVA.ovaをダブルクリックして`VMware Player`にOVAをインポートします。
コンバートしたOVAが正常に動作することを確認できました。

### IDCFクラウドにデプロイ

作成したAtomicHostOVA.ovaを、OVAテンプレートとして新規追加してからインスタンスを作成します。


### Atomic HostインスタンスにSSH接続

OVAは予め作業ユーザーとSSH公開鍵を設定してるので、SSH接続できる状態になっています。

``` bash
$ eval `ssh-agent`
$ ssh-add .ssh/mykey
$ ssh -A 10.1.0.56
```

OSのリリースを確認します。

``` bash
$ cat /etc/redhat-release
Generic release 20 (Generic)
```

Dockerのバージョンです。

``` bash
$ docker version
Client version: 1.0.0
Client API version: 1.12
Go version (client): go1.2.2
Git commit (client): 63fe64c/1.0.0
Server version: 1.0.0
Server API version: 1.12
Go version (server): go1.2.2
Git commit (server): 63fe64c/1.0.0
```

dockerのunitを確認すると、`--selinux-enabled`になっています。

``` bash
$ systemctl status docker
docker.service - Docker Application Container Engine
   Loaded: loaded (/usr/lib/systemd/system/docker.service; enabled)
   Active: active (running) since 土 2014-09-13 04:12:50 UTC; 5h 57min ago
     Docs: http://docs.docker.io
 Main PID: 477 (docker)
   CGroup: /system.slice/docker.service
           └─477 /usr/bin/docker -d --selinux-enabled -H fd://
```

### 確認

`hello world`でコンテナの起動を確認します。

``` bash
$ sudo docker run fedora /bin/echo "hello world"
Unable to find image 'fedora' locally
Pulling repository fedora
88b42ffd1f7c: Download complete
511136ea3c5a: Download complete
c69cab00d6ef: Download complete
hello world
```