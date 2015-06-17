title: "Flocker with ZFS on IDCFクラウド- Part1: ZFS on Linuxインストール"
date: 2014-08-21 00:25:49
tags:
 - Docker管理
 - Flocker
 - Fedora20
 - ZFS
 - geard
 - Deis
 - IDCFクラウド
 - DataVolumeContainer
description: Deisからdeis config:setして使うデータベース用のコンテナをどう管理しようか考えていたところ、ZFSベースのDockerコンテナ管理ツール「Flocker」という記事で、Flockerというデータボリュームに特化したDockerコンテナ管理ツールを見つけました。ZFS on Linuxを使った独自のZFSレプリケーションが特徴です。数年前にちょっと触ったSmartOSみたい。FlockerはTwistedのコアメンバーが在籍してるClusterHQがオープンソースで開発しています。Getting StartedではVagrantと構成済みのFedora20のBoxを使うことを推奨しています。でもZoLの方が気になってしまい、IDCFクラウド上にFedora20のインスタンスを用意してFlocker用にZFSとgeardをマニュアルインストールしてみます。まずはZFSのインストールとストレージプールを作成します。
---

Deisから`deis config:set`して使うデータベース用のコンテナをどう管理しようか考えていたところ、
[ZFSベースのDockerコンテナ管理ツール「Flocker」](http://sourceforge.jp/magazine/14/08/15/163000)という記事で、[Flocker](https://github.com/ClusterHQ/flocker)というデータボリュームに特化したDockerコンテナ管理ツールを見つけました。

[ZFS on Linux](http://zfsonlinux.org/)を使った独自のZFSレプリケーションが特徴です。数年前にちょっと触った[SmartOS](http://smartos.org/)みたい。

Flockerは[Twisted](https://twistedmatrix.com/)のコアメンバーが在籍してる[ClusterHQ](https://clusterhq.com/)がオープンソースで開発しています。

[Getting Started](https://docs.clusterhq.com/en/0.1.0/gettingstarted/)ではVagrantと構成済みのFedora20のBoxを使うことを推奨しています。でもZoLの方が気になってしまい、IDCFクラウド上にFedora20のインスタンスを用意してFlocker用にZFSとgeardをマニュアルインストールしてみます。

まずはZFSのインストールとストレージプールを作成します。

<!-- more -->

### Fedora20インスタンスの用意

`--ostypeid`はFedora13を指定して、IDCFクラウドにFedora20のISOを登録します。

``` bash
$ idcf-compute-api registerIso \
  --name=fedora20_x86_64  \
  --displaytext=fedora20_x86_64 \
  --url=http://download.fedoraproject.org/pub/fedora/linux/releases/20/Live/x86_64/Fedora-Live-Desktop-x86_64-20-1.iso \
  --zoneid=1 \
  --ostypeid=115 \
  --bootable=true
```

ポータル画面からディスクを100GBに指定して、ISOからインスタンスを作成します。

コンソール画面からFedora20を肯定的にインストールした後、SSHのインストールを行います。

``` bash
$ sudo yum install openssh
$ sudo systemctl start sshd.service
$ sudo systemctl enable sshd.service
```

SSHの設定変更をします。

``` bash
$ sudo vi /etc/ssh/sshd_config
PermitRootLogin no
PasswordAuthentication no
$ sudo systemctl reload sshd.service
```

これでリモートの作業マシンからパスワードでSSH接続できるようになりました。

### ZFSのインストール

Fedora20インスタンスに作業マシンからSSH接続して、zfsのリポジトリを追加後にZFSをインストールします。

``` bash
$ sudo yum localinstall --nogpgcheck http://archive.zfsonlinux.org/fedora/zfs-release$(rpm -E %dist).noarch.rpm
$ sudo yum install zfs
```

`yum update`してからrebootします。

``` bash
$ sudo yum update
$ sudo reboot
```

### テストのためストレージプール作成

とりあえずテスト的に1Gで`/opt`にストレージプールを作成してみます。

``` bash
$ sudo mkdir -p /opt/flocker
$ sudo truncate --size 1G /opt/flocker/pool-vdev
$ sudo zpool create flocker /opt/flocker/pool-vdev
$ sudo zfs list
NAME      USED  AVAIL  REFER  MOUNTPOINT
flocker   108K   984M    30K  /flocker
```

`df`をしてみると、100GBのディスクでデフォルトのパーティションの場合、`/home`に50GBが割り当てられています。

``` bash
$ df -h
ファイルシス                              サイズ  使用  残り 使用% マウント位置
/dev/mapper/fedora_i--669--69116--vm-root    50G  4.0G   43G    9% /
devtmpfs                                    2.0G     0  2.0G    0% /dev
tmpfs                                       2.0G   80K  2.0G    1% /dev/shm
tmpfs                                       2.0G  792K  2.0G    1% /run
tmpfs                                       2.0G     0  2.0G    0% /sys/fs/cgroup
tmpfs                                       2.0G   16K  2.0G    1% /tmp
/dev/sda1                                   477M  115M  334M   26% /boot
/dev/mapper/fedora_i--669--69116--vm-home    45G   57M   43G    1% /home
flocker                                     984M     0  984M    0% /flocker
```

### ストレージプールを作成し直す

zpoolのステータスを確認します。

``` bash
$ sudo zpool status
  pool: flocker
 state: ONLINE
  scan: none requested
config:

        NAME                      STATE     READ WRITE CKSUM
        flocker                   ONLINE       0     0     0
          /opt/flocker/pool-vdev  ONLINE       0     0     0

errors: No known data errors
```

zpoolを破棄します。

``` bash
$ sudo zpool destroy flocker
$ sudo zpool status
no pools available
```

`/home`ディレクトリにマントポイントを再作成します。

``` bash
$ sudo mkdir -p /home/flocker
$ sudo truncate --size 40G /home/flocker/pool-vdev
$ sudo zpool create flocker /home/flocker/pool-vdev
```

### 確認

`df`で確認すると、`/flocker`に40GBのストレージプールが作成できました。

``` bash
$ df -h
ファイルシス                              サイズ  使用  残り 使用% マウント位置
/dev/mapper/fedora_i--669--69116--vm-root    50G  4.0G   43G    9% /
devtmpfs                                    2.0G     0  2.0G    0% /dev
tmpfs                                       2.0G   80K  2.0G    1% /dev/shm
tmpfs                                       2.0G  788K  2.0G    1% /run
tmpfs                                       2.0G     0  2.0G    0% /sys/fs/cgroup
tmpfs                                       2.0G   16K  2.0G    1% /tmp
/dev/sda1                                   477M  115M  334M   26% /boot
/dev/mapper/fedora_i--669--69116--vm-home    45G   58M   43G    1% /home
flocker                                      40G     0   40G    0% /flocker
```

`zfs list`でも確認しておきます。

``` bash
$ sudo zfs list
NAME      USED  AVAIL  REFER  MOUNTPOINT
flocker   108K  39.1G    30K  /flocker
```
