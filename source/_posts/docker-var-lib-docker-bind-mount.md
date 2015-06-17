title: "Dockerの/var/lib/dockerを移動する"
date: 2015-06-02 21:56:48
tags:
 - Docker
 - DockerCompose
description: 今使っているクラウドのルートディスクが15GBしかないのでDocker Composeで複数のコンテナを起動して開発しているとすぐに容量不足になります。開発中はよくゴミが出るので定期的に/var/lib/dockerは再作成していますが、もう少し大きなディスクを追加して移動しようと思います。最初は/etc/fstabの記述を間違えてリカバリーモードで起動することになりました。しかもrootのパスワードを忘れてしまいシングルモードで起動する必要があったりと、作業には十分注意が必要です。以下のサイトを参考にしました。最初に読んでいたサイトはmountのbindオプションの使い方が間違っていて混乱してしまいました。Moving docker images location to different partition Moving Docker "stuff" to a Different Drive
---

今使っているクラウドのルートディスクが15GBしかないのでDocker Composeで複数のコンテナを起動して開発しているとすぐに容量不足になります。開発中はよくゴミが出るので定期的に`/var/lib/docker`は再作成していますが、もう少し大きなディスクを追加して移動しようと思います。最初は`/etc/fstab`の記述を間違えてリカバリーモードで起動することになりました。しかもrootのパスワードを忘れてしまいシングルモードで起動する必要があったりと、作業には十分注意が必要です。

以下のサイトを参考にしました。最初に読んでいたサイトはmountのbindオプションの使い方が間違っていて混乱してしまいました。

* [Moving docker images location to different partition](http://alexander.holbreich.org/2014/07/moving-docker-images-different-partition/)
* [Moving Docker "stuff" to a Different Drive](http://peterjolson.com/moving-docker-stuff-to-a-different-drive/)

<!-- more -->

## 環境のバージョン

最初に作業環境のDockerやOSのバージョンを確認します。

Dockerのバージョン

``` bash
$ docker version
Client version: 1.6.2
Client API version: 1.18
Go version (client): go1.4.2
Git commit (client): 7c8fca2
OS/Arch (client): linux/amd64
Server version: 1.6.2
Server API version: 1.18
Go version (server): go1.4.2
Git commit (server): 7c8fca2
OS/Arch (server): linux/amd64
```

カーネル

```bash
$ cat /proc/version
Linux version 3.13.0-33-generic (buildd@tipua) (gcc version 4.8.2 (Ubuntu 4.8.2-19ubuntu1) ) #58-Ubuntu SMP Tue Jul 29 16:45:05 UTC 2014
```

Ubuntu 14.04.2

```bash
$ cat /etc/lsb-release
NAME="Ubuntu"
VERSION="14.04.2 LTS, Trusty Tahr"
ID=ubuntu
ID_LIKE=debian
PRETTY_NAME="Ubuntu 14.04.2 LTS"
VERSION_ID="14.04"
HOME_URL="http://www.ubuntu.com/"
SUPPORT_URL="http://help.ubuntu.com/"
BUG_REPORT_URL="http://bugs.launchpad.net/ubuntu/"
```

## ディスクの追加

作業前のディスクの使用状況です。

```bash
$ df -h
Filesystem      Size  Used Avail Use% Mounted on
/dev/sda1        15G  4.1G  9.9G  29% /
none            4.0K     0  4.0K   0% /sys/fs/cgroup
udev            2.0G  4.0K  2.0G   1% /dev
tmpfs           395M  816K  394M   1% /run
none            5.0M     0  5.0M   0% /run/lock
none            2.0G  476K  2.0G   1% /run/shm
none            100M     0  100M   0% /run/user
```

40GBのディスクを追加した状態です。今回はクラウド上ですがfdiskでデバイスが認識されたことがわかります。まだパーティションありません。

```bash
$ sudo fdisk -l

Disk /dev/sda: 16.1 GB, 16106127360 bytes
255 heads, 63 sectors/track, 1958 cylinders, total 31457280 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk identifier: 0x000db9c4

   Device Boot      Start         End      Blocks   Id  System
/dev/sda1   *        2048    31455231    15726592   83  Linux

Disk /dev/sdb: 42.9 GB, 42949672960 bytes
255 heads, 63 sectors/track, 5221 cylinders, total 83886080 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk identifier: 0x00000000

Disk /dev/sdb doesn't contain a valid partition table
```

### パーティションの作成

fdiskでパーティションを作成します。コマンドとしては`n、p、[Enter]x3回、w`をキーを入力します。

``` bash
$ sudo fdisk /dev/sdb
Device contains neither a valid DOS partition table, nor Sun, SGI or OSF disklabel
Building a new DOS disklabel with disk identifier 0xef0c9f8a.
Changes will remain in memory only, until you decide to write them.
After that, of course, the previous content won't be recoverable.

Warning: invalid flag 0x0000 of partition table 4 will be corrected by w(rite)

Command (m for help): n
Partition type:
   p   primary (0 primary, 0 extended, 4 free)
   e   extended
Select (default p): p
Partition number (1-4, default 1):
Using default value 1
First sector (2048-83886079, default 2048):
Using default value 2048
Last sector, +sectors or +size{K,M,G} (2048-83886079, default 83886079):
Using default value 83886079

Command (m for help): w
The partition table has been altered!

Calling ioctl() to re-read partition table.
Syncing disks.
```

パーティション作成の確認をします。`/dev/sdb1`が作成されました。

``` bash
$ sudo fdisk -l


Disk /dev/sda: 16.1 GB, 16106127360 bytes
255 heads, 63 sectors/track, 1958 cylinders, total 31457280 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk identifier: 0x000db9c4

   Device Boot      Start         End      Blocks   Id  System
/dev/sda1   *        2048    31455231    15726592   83  Linux

Disk /dev/sdb: 42.9 GB, 42949672960 bytes
171 heads, 5 sectors/track, 98112 cylinders, total 83886080 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk identifier: 0xef0c9f8a

   Device Boot      Start         End      Blocks   Id  System
/dev/sdb1            2048    83886079    41942016   83  Linux
```

### ファイルシステムの作成

既存ファイルシステムはext4なので同じタイプで作成します。

``` bash
$ df -T
Filesystem     Type     1K-blocks    Used Available Use% Mounted on
/dev/sda1      ext4      15348720 4212380  10333628  29% /
none           tmpfs            4       0         4   0% /sys/fs/cgroup
udev           devtmpfs   2009748       4   2009744   1% /dev
tmpfs          tmpfs       404100     820    403280   1% /run
none           tmpfs         5120       0      5120   0% /run/lock
none           tmpfs      2020484     476   2020008   1% /run/shm
none           tmpfs       102400       0    102400   0% /run/user
```

ext4のファイルシステムを作成します。

```bash
$ sudo mkfs.ext4 /dev/sdb1
mke2fs 1.42.9 (4-Feb-2014)
Filesystem label=
OS type: Linux
Block size=4096 (log=2)
Fragment size=4096 (log=2)
Stride=0 blocks, Stripe width=0 blocks
2621440 inodes, 10485504 blocks
524275 blocks (5.00%) reserved for the super user
First data block=0
Maximum filesystem blocks=4294967296
320 block groups
32768 blocks per group, 32768 fragments per group
8192 inodes per group
Superblock backups stored on blocks:
	32768, 98304, 163840, 229376, 294912, 819200, 884736, 1605632, 2654208,
	4096000, 7962624

Allocating group tables: done
Writing inode tables: done
Creating journal (32768 blocks): done
Writing superblocks and filesystem accounting information: done
```

## パーティションのマウント

新しくパーティションをマウントしてDockerの`/var/lib/docker`を移動していきます。まずDockerを停止します。

``` bash
$ sudo service docker stop
docker stop/waiting
```

パーティションをマウントするディレクトリを作成します。Ubuntuは`/media`に作ることが多いです。

```bash
$ sudo mkdir /media/data
```

`/etc/fstab`の編集は危険なので必ずパックアップをとります。

``` bash
$ sudo cp -ip /etc/fstab{,.`date +%Y%m%d`}
```

パーティションの UUID を`blkid`コマンドで確認します。

```bash
$ sudo  blkid | grep sdb1
/dev/sdb1: UUID="d45eaaab-e223-4162-96b3-fefea9392cad" TYPE="ext4"}}}
```

`/etc/fstab`に新しいパーティションのUUIDを追加します。

```bash /etc/fstab
UUID=ed7bdb81-136d-458b-8121-2c4e5d3f569f /               ext4    errors=remount-ro 0       1
/dev/fd0        /media/floppy0  auto    rw,user,noauto,exec,utf8 0       0

UUID=d45eaaab-e223-4162-96b3-fefea9392cad /media/data ext4 defaults 0  2
```

いつも忘れるのですが、4,5,6列目は以下のように設定しました。

* 4列目: デフォルト
* 5列目: 0: dumpする
* 6列目: 2: fsckする

`/etc/fstab`の設定を再マウントします。

```bash
$ sudo mount -a
```

`/dev/sdb1`のデバイスがマウントされました。

```bash
$ df -h
Filesystem      Size  Used Avail Use% Mounted on
/dev/sda1        15G  4.1G  9.9G  29% /
none            4.0K     0  4.0K   0% /sys/fs/cgroup
udev            2.0G  8.0K  2.0G   1% /dev
tmpfs           395M  792K  394M   1% /run
none            5.0M     0  5.0M   0% /run/lock
none            2.0G     0  2.0G   0% /run/shm
none            100M     0  100M   0% /run/user
/dev/sdb1        40G   48M   38G   1% /media/data
```

## /var/lib/dockerの移動

### rsyncでディレクトリをコピー

`rsync`で`/var/lib/docker`のファイルを新しいディレクトリにコピーします。

``` bash
$ sudo mkdir /media/data/docker
$ sudo rsync -aXS /var/lib/docker/. /media/data/docker/
```

`rsync`が終了して、`/var/lib/docker`と`/media/data/docker/`が同じ内容になりました。

```bash
$ sudo du -sch /var/lib/docker/*
2.1G	/var/lib/docker/aufs
140K	/var/lib/docker/containers
808K	/var/lib/docker/graph
15M	/var/lib/docker/init
12K	/var/lib/docker/linkgraph.db
4.0K	/var/lib/docker/repositories-aufs
4.0K	/var/lib/docker/tmp
8.0K	/var/lib/docker/trust
44K	/var/lib/docker/volumes
2.2G	total
$ sudo du -sch /media/data/docker/*
2.1G	/media/data/docker/aufs
140K	/media/data/docker/containers
808K	/media/data/docker/graph
15M	/media/data/docker/init
12K	/media/data/docker/linkgraph.db
4.0K	/media/data/docker/repositories-aufs
4.0K	/media/data/docker/tmp
8.0K	/media/data/docker/trust
44K	/media/data/docker/volumes
2.1G	total
```


## ディレクトリのマウント

オリジナルの`/var/lib/docker`を念のためリネームしてバックアップします。その後ディレクトリを空にして作り直します。

```bash
$ sudo mv /var/lib/docker /var/lib/docker.bak
$ sudo mkdir /var/lib/docker
```

`/var/lib/docker`は空のディレクトリの状態です。

```bash
$ ls /media/data/docker/
aufs        graph  linkgraph.db       tmp    volumes
containers  init   repositories-aufs  trust
$ ls /var/lib/docker
```

`/media/data/docker`を`/var/lib/docker`にbindオプションを付けてマウントします。[BINDオプションでのmount](http://wiki.bit-hive.com/north/pg/BIND%A5%AA%A5%D7%A5%B7%A5%E7%A5%F3%A4%C7%A4%CEmount)に詳しく解説してくれています。mountのbindオプションを使うと任意のディレクトリを独自のファイルシステムとしてアタッチすることができます。ディレクトリのあるデバイスのパーティションがマウントされている必要があります。

```bash
$ sudo mount -o bind  /media/data/docker /var/lib/docker
$ sudo du -sch /var/lib/docker/*
2.1G	/var/lib/docker/aufs
140K	/var/lib/docker/containers
808K	/var/lib/docker/graph
15M	/var/lib/docker/init
12K	/var/lib/docker/linkgraph.db
4.0K	/var/lib/docker/repositories-aufs
4.0K	/var/lib/docker/tmp
8.0K	/var/lib/docker/trust
44K	/var/lib/docker/volumes
2.1G	total
```

`/media/data/docker`ディレクトリが`/var/lib/docker`にマウントされました。最初はこれをfstabに逆で書いてしてしまい起動しなくなってしまいました。

```bash
$ df -ha
Filesystem       Size  Used Avail Use% Mounted on
/dev/sda1            15G  4.1G  9.9G  29% /
...
/dev/sdb1            40G  2.2G   36G   6% /media/data
/media/data/docker   40G  2.2G   36G   6% /var/lib/docker
```

`/etc/fstab`を編集して再起動後も有効にします。

```bash /etc/fstab
/media/data/docker /var/lib/docker  none bind 0 0
```

mountをリロードします。

``` bash
$ sudo mount -a
```

## 移動先からDockerを起動して確認

### aufsだけ古いディレクトリで増加してしまう

Dockerを起動します。

```bash
$ sudo service docker start
docker start/running, process 4276
```

`/var/lib/docker/aufs`だけ4.3Gに増加してしまいました。 `/media/data/docker/aufs`は変化していません。aufsは以前の場所のままのようです。この状態で新しいイメージを`docker pull`しても`/var/lib/docker`は増加しません。

```bash
$ sudo du -sch /var/lib/docker/*
4.3G	/var/lib/docker/aufs
168K	/var/lib/docker/containers
808K	/var/lib/docker/graph
15M	/var/lib/docker/init
12K	/var/lib/docker/linkgraph.db
4.0K	/var/lib/docker/repositories-aufs
4.0K	/var/lib/docker/tmp
8.0K	/var/lib/docker/trust
44K	/var/lib/docker/volumes
4.3G	total
$ sudo du -sch /media/data/docker/*
2.1G	/media/data/docker/aufs
168K	/media/data/docker/containers
808K	/media/data/docker/graph
15M	/media/data/docker/init
12K	/media/data/docker/linkgraph.db
4.0K	/media/data/docker/repositories-aufs
4.0K	/media/data/docker/tmp
8.0K	/media/data/docker/trust
44K	/media/data/docker/volumes
2.1G	total
```

aufsが増えた原因は、docker-compose.ymlのサービスを`restart: always`にしているのでDockerの起動時に開始したためです。すでに作成済みのコンテナはmountのbindオプションだけではうまく移動できないみたいです。

```bash
$ docker-compose ps
      Name             Command             State              Ports
-------------------------------------------------------------------------
meshblucompose_m   npm start          Up                 0.0.0.0:1883->18
eshblu_1                                                 83/tcp, 9000/tcp
meshblucompose_m   /entrypoint.sh     Up                 27017/tcp
ongo_1             mongod
meshblucompose_o   nginx -c /etc/ng   Up
penresty_run_1     inx/nginx. ...
meshblucompose_r   /entrypoint.sh     Up                 6379/tcp
edis_1             redis-server
```

### DOCKER_OPTSに追加する

Dockerのオプションに新しい`/var/lib/docker`のディレクトリを指定する必要がありました。Dockerを停止してから設定ファイルに追記します。

```bash /etc/default/docker
DOCKER_OPTS="-g /media/data/docker"
```

Dockerを起動します。

``` bash
$ sudo service docker start
docker start/running, process 5786
```

ようやく`/var/lib/docker`が増加せず、`/media/data/docker/`のディレクトリのaufsが増加しました。`/var/lib/docker`の移動ができました。

```bash
$ sudo du -sch /var/lib/docker/*
2.1G	/var/lib/docker/aufs
180K	/var/lib/docker/containers
808K	/var/lib/docker/graph
15M	/var/lib/docker/init
12K	/var/lib/docker/linkgraph.db
4.0K	/var/lib/docker/repositories-aufs
4.0K	/var/lib/docker/tmp
8.0K	/var/lib/docker/trust
44K	/var/lib/docker/volumes
2.1G	total
$ sudo du -sch /media/data/docker/*
4.3G	/media/data/docker/aufs
180K	/media/data/docker/containers
808K	/media/data/docker/graph
15M	/media/data/docker/init
12K	/media/data/docker/linkgraph.db
4.0K	/media/data/docker/repositories-aufs
4.0K	/media/data/docker/tmp
8.0K	/media/data/docker/trust
44K	/media/data/docker/volumes
4.3G	total
```
