title: "Project Atomic First Steps- Part1: Atomic Hostのビルド"
date: 2014-09-14 01:39:17
tags:
 - ProjectAtomic
 - Fedora20
 - VMwarePlayer
 - IDCFクラウド
 - config-drive
 - OVA
 - Minnty
 - ovftool
description: OpenShift Origin 3とKubernetesを試していると、同じRedHatのProject Atomicも気になりAtomic Host上で動かしたくなります。イメージはGet Started with Atomicからダウンロードできるので、VirtualBoxですぐに試すことができます。Atomic HostはCoreOSと比較されSELinuxとrpm-ostree,geardが特徴です。Fedoraはほとんど使ったことがないのでCoreOS vs. Project Atomic A Reviewを読みながら勉強していきます。
---

`OpenShift Origin 3`とKubernetesを試していると、同じRedHatの[Project Atomic](http://www.projectatomic.io/)も気になり`Atomic Host`上で動かしたくなります。イメージは[Get Started with Atomic](http://www.projectatomic.io/download/)からダウンロードできるので、VirtualBoxですぐに試すことができます。

`Atomic Host`はCoreOSと比較され、SELinuxと[rpm-ostree](https://github.com/projectatomic/rpm-ostree)が[geard](https://github.com/openshift/geard)特徴です。Fedoraはほとんど使ったことがないので[CoreOS vs. Project Atomic: A Review](https://major.io/2014/05/13/coreos-vs-project-atomic-a-review/)を読みながら勉強していきます。


<!-- more -->


### rpm-ostreeをちょっと調べた

rpm-ostreeは、RPMパッケージで構成されたOSを`git fetch`と`git merge`のように管理できる仕組みです。
中央リポジトリからアップグレードをfetch && mergeして、reboot後は新しい/rootで起動する感じです。
Gitのように新しいOSの複製と配布がとても簡単になり、この辺りが`atomic`らしいです。


* [Build Your Own Atomic Image, Updated](http://www.projectatomic.io/blog/2014/08/build-your-own-atomic-centos-or-fedora/)

* [CentOS 7 Alpha Builds for Atomic](http://www.projectatomic.io/blog/2014/08/centos-7-alpha-builds-for-atomic/)

### Fedora20の準備

[CentOS 7](http://www.projectatomic.io/blog/2014/08/build-your-own-atomic-centos-or-fedora/)でも`Atomic Host`のビルドができるようになったそうですが、手順通りにFedora20で作業することにします。

IDCFクラウドの標準テンプレートにはFedora20がないので、ISOをアップロードしてからインスタンスを作成します。

### SELinuxをdisabled

イメージのビルド作業のために、SELinuxをdisabledにします。

``` bash /etc/selinux/config
#SELINUX=enforcing
SELINUX=disabled
```

一度rebootします。

``` bash
# reboot
```

### パッケージのダウンロード


`Atomic Host`ビルド用リポジトリを`git clone`し、必要なパッケージをインストールします。

``` bash
# yum install -y git
# git clone https://github.com/jasonbrooks/byo-atomic.git
# mv byo-atomic/walters-rpm-ostree-fedora-20-i386.repo /etc/yum.repos.d/
# yum install -y rpm-ostree rpm-ostree-toolbox nss-altfiles yum-plugin-protectbase httpd
```

### 設定ファイルの編集

/etc/nsswitch.confのバックアップをします。

``` bash
# cp /etc/nsswitch.conf{,.orig}
```

`passwd:`と`group:`の行を編集します。

``` bash /etc/nsswitch.conf
#passwd:     files
passwd:     files altfiles
shadow:     files
#group:      files
group:      files altfiles
```

/etc/libvirt/qemu.confのバックアップをします。

``` bash
# cp /etc/libvirt/qemu.conf{,.orig}
```

`user = "root"`の行をアンコメントします。

``` bash /etc/libvirt/qemu.conf 
user = "root"
```

libvirtdをrestartします。

``` bash
# systemctl restart libvirtd
```

### rpm-ostreeのリポジトリ

`Atomic Host`のRPMをアップデートするため、rpm-ostreeのリポジトリを作成します。

``` bash
# mkdir -p /srv/rpm-ostree/repo && cd /srv/rpm-ostree/ && sudo ostree --repo=repo init --mode=archive-z2
# cat > /etc/httpd/conf.d/rpm-ostree.conf <<EOF
DocumentRoot /srv/rpm-ostree
<Directory "/srv/rpm-ostree">
Options Indexes FollowSymLinks
AllowOverride None
Require all granted
</Directory>
EOF
# systemctl daemon-reload && \
  systemctl enable httpd && \
  systemctl start httpd && \
  systemctl reload httpd && \
  firewall-cmd --add-service=http && \
  firewall-cmd --add-service=http --permanent
```

### Atomic Hostイメージビルド

`Atomic Host`イメージをビルドします。rpm-ostree-toolboxを使いqcow2ファイルを作成します。

``` bash
# cd /root/byo-atomic/f20
# rpm-ostree compose tree --repo=/srv/rpm-ostree/repo fedora-atomic-server-docker-host.json
# rpm-ostree-toolbox create-vm-disk /srv/rpm-ostree/repo fedora-atomic-host fedora-atomic/20/x86_64/server/docker-host f20-atomic.qcow2
...
Started child process 'guestunmount' '-v' '/tmp/rpmostreetoolbox.YG4AMX/mnt': pid=24463
Awaiting termination of guestmount, watching: /tmp/rpmostreetoolbox.YG4AMX/mnt.guestmount-pid
guestmount pid file successfully deleted
Created: /root/byo-atomic/f20/f20-atomic.qcow2
```

### vmdkファイルにコンバート

qemu-imgコマンドを使い、qcow2ファイルをvmdkファイルにコンバートします。

``` bash
# qemu-img convert -f qcow2 /root/byo-atomic/f20/f20-atomic.qcow2 -O vmdk c7-atomic.vmdk
```

### config-driveの作成

config-driveは仮想マシン起動時に読み込んで利用する、OSの設定情報を格納したISOファイルです。
ISOファイルを仮想マシン起動時に接続すると、`meta-data`と`user-data`を読み込み設定をしてくれます。

`Atomic Host`はrootのパスワードや公開鍵を記述します。この辺りはCoreOSと同じです。

`meta-data`にはホスト名を指定します。

``` bash meta-data
instance-id: Atomic0
local-hostname: atomic-00
```

`user-data`にはパスワードと公開鍵を指定します。

``` bash user-data
#cloud-config
password: atomic
chpasswd: {expire: False}
ssh_pwauth: True
ssh_authorized_keys
  - ssh-rsa AAAAB3NzaC1yc...
```

genisoimageコマンドを使いISOファイルを作成します。

``` bash
$ genisoimage -output atomic0-cidata.iso -volid cidata -joliet -rock -input-charset utf-8 user-data meta-data
Total translation table size: 0
Total rockridge attributes bytes: 331
Total directory bytes: 0
Path table size(bytes): 10
Max brk space used 0
183 extents written (0 MB)
```

### VMware Playerでvmdkをアタッチして起動

Fedora20で作成したc7-atomic.vmdkファイルを、Window7の作業マシンにダウンロードします。
以下の手順で仮想マシンを作成します。ディスクイメージを作成したvmdkファイルと入れ替えます。

01. `VMware Player`を起動
02. 新規仮想マシンの作成
03. 後でOSをインストールを選択
04. Linux, 他のLinux 3.xカーネル 64ビットを選択
05. 仮想マシン名に、AtomicHost
06. 1GBで、仮想ディスクを単一ファイルとして格納
07. c7-atomic.vmdkを`Virtual Machines`のAtomicHostフォルダにコピー
08. 仮想マシンの設定の編集
09. 既存のハードディスクを削除
10. 既存の仮想ディスクを使用からc7-atomic.vmdkを選択、既存の形式を保持
11. `config-drive`のISOファイルをCD/DVD(IDE)にセットし、起動時に接続にチェックを入れる
12. 仮想マシンを起動


### VMware Playerコンソール

`VMware Player`コンソールから`Atomic Host`の仮想マシンにログインします。

 * login: fedora
 * passwd: atomic

パスワードはconfig-driveに設定した値を使います。

### IDCFクラウドではconfig-driveが使えない

ovftoolを使いOVAにコンバートしてからIDCFクラウドのテンプレートに登録します。
OVAテンプレートからVMを作成するときに、config-driveのISOファイルをマウントします。

残念ながらIDCFクラウドでは仮想マシンの起動時にconfig-driveを読み込んでくれません。
`user-data`を取得できずにOSの設定が完了しないためログインできなくなります。

### まとめ

`Atomic Host`をビルドしながら`rpm-ostree`の役割や`rpm-ostree-toolbox`コマンドを使ったディスクイメージの作り方など勉強することができました。

`config-drive`も便利な仕組みです。仮想マシン起動時に`user-data`を渡してOSの構成管理を初回で終わらせてしまう方法はCoreOSと同じです。
`cloud-config`でOSを作成した後のパッケージ管理は中央リポジトリと同期され、個別に設定ファイルの書き換えやパッケージの追加ができなくなります。`Project Atomic`ではyumの廃止やそれに伴いPythonもインストールしないことも検討されているようです。

Webスケールで使い捨て可能なインフラが必要な環境では、ChefなどのCMツールを使わずに`cloud-config`を使った構成管理が今後主流になると思うので慣れておきたいです。

次回はビルド済みの`Atomic Host`のダウンロードを使ってIDCFクラウドにデプロイしてみます。
