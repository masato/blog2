title: 'BeagleBone Blackで1-WireセンサのDS18B20を使う - Part1: SDカードのUbuntuの復旧'
date: 2015-07-09 23:17:04
categories:
 - IoT
tags:
 - DS18B20
 - BeagleBoneBlack
 - Ubuntu
 - Debian
description: ことの始まりは1-WireセンサのDS18B20をBeagleBone Blackに接続しようとしたことです。SDカードにインストールしていたUbuntu 14.04はカーネルが3.14と新しくCapemgrが存在していません。dtb-rebuilderを使ってDTSを編集してビルドした後、/boot/uEnv.txtのDTBを書き換える必要があります。ところが書き間違えてSDカードからbootしなくなりました。なんとかSDカードは復旧したのですが、カーネルが3.8.13のDebian 7.8を入れ直した方がよさそうです。
---


ことの始まりは1-Wireセンサの[DS18B20](http://victory7.com/?pid=65664796)をBeagleBone Blackに接続しようとしたことです。SDカードにインストールしていたUbuntu 14.04はカーネルが3.14と新しく[Capemgr](http://elinux.org/Capemgrが存在していません。[dtb-rebuilder](https://github.com/RobertCNelson/dtb-rebuilder)を使って[DTS](https://github.com/RobertCNelson/dtb-rebuilder/tree/3.14-ti/src/arm)を編集してビルドした後、/boot/uEnv.txtのDTBを書き換える必要があります。ところが書き間違えてSDカードからbootしなくなりました。なんとかSDカードは復旧したのですが、カーネルが3.8.13のDebian 7.8を入れ直した方がよさそうです。

<!-- more -->

## eMMCのDebianからSDカードの復旧

SDカードのUbuntu 14.04はDebian 7.8に入れ替えますが、備忘のため`/boot/uEnv.txt`を書き間違えてbootしなくなったSDカードの復旧についてメモしておきます。まずSDカードを抜いてeMMCからbootします。この場合はDebianが入っています。起動後にSDカードを挿します。fdiskで確認すると、SDカードは`/dev/mmcblk1`のデバイスのようです。

```bash
$ sudo fdisk -l
...
        Device Boot      Start         End      Blocks   Id  System
/dev/mmcblk1p1   *        2048     3481599     1739776   83  Linux
Disk /dev/mmcblk1: 15.7 GB, 15720251392 bytes
```

lsblkでデバイス名とmount pointも確認します。

```bash
$ lsblk
NAME         MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
mmcblk0boot0 179:8    0     1M  1 disk
mmcblk0boot1 179:16   0     1M  1 disk
mmcblk0      179:0    0   1.8G  0 disk
├─mmcblk0p1  179:1    0    96M  0 part /boot/uboot
└─mmcblk0p2  179:2    0   1.7G  0 part /
mmcblk1      179:24   0  14.7G  0 disk
└─mmcblk1p1  179:25   0   1.7G  0 part /media/rootfs
```

SDカードにある`/boot/uEnv.txt`が見つかりました。

```bash
$ ls -al /media/rootfs/boot/
-rw-r--r--  1 root root     414 Jan  6  2015 uEnv.txt
```

あとは間違えて設定した`dtb=`の記述を消します。

```bash
$ sudo vi /media/rootfs/boot/uEnv.txt
```

SDカードを挿したままrebootすれば、SDカードのUbuntuが起動するようになりました。

```bash
$ sudo reboot
```
