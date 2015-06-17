title: 'Packerを使いWindows上でOVAを作成する - Part8: Nested VirtualBox デスクトップ環境のディスク拡張'
date: 2014-06-15 19:46:49
tags:
 - PackerOVA
 - VirtualBox
 - VBoxManage
 - Vagrant
 - Vagrantデスクトップ
 - Packer
 - Ubuntu
 - Mintty
 - LVM
 - NestedVirtualization
description: Part6とPart7でVirtualBox 4.3.12上にESXi5の仮想マシンを作成して、remote buildを実行できるように環境構築しました。ESXi5でVNCのインバウンドのFW設定も入れましたが、ゲストVMにVNC接続できず、Preseedが実行できませんでした。なぜかPart2でWindowsのVMWareWorkstationの環境ではPackerでOVA作成に成功しています。作業環境でPreseedの実行をVNCで確認できた方がデバッグしやすいので、VagrantでUbuntu14.04のデスクトップ環境を構築するところから始めます。デフォルトのディスクサイズが小さいので、デスクトップ環境ができたら、VMware Player￥をインストールして、PackerからOVAを作成してみます。
---
[PackerでOVA作成](http://masato.github.io/tags/PackerOVA/)シリーズです。
[Part6](/2014/06/13/packer-windows-vagrant-remote-esxi5-build/)[Part7](/2014/06/14/packer-windows-vagrant-remote-esxi5-partedUtil)で`VirtualBox上 4.3.12`にESXi5の仮想マシンを作成して、`remote build`を実行できるように環境構築をしました。ESXi5でVNCのインバウンドのFW設定も入れましたが、ゲストVMにVNC接続できず、Preseedが実行できませんでした。

なぜか[Part2](/2014/06/08/packer-windows-vmware-iso-ova-build/)でWindowsのVMWareWorkstationの環境ではPackerでOVA作成に成功しています。

作業環境でPreseedの実行をVNCで確認できた方がデバッグしやすいので、VagrantでUbuntu14.04のデスクトップ環境を構築するところから始めます。デフォルトのディスクサイズが小さいので、

デスクトップ環境ができたら、`VMware Player`をインストールして、PackerからOVAを作成してみます。

<!-- more -->

### デスクトップ用のBoxを探す

VagrantのBoxは、[Vagrant Cloud](https://vagrantcloud.com/discover/featured)から検索します。
[ubuntu desktop 14.05](https://vagrantcloud.com/search?utf8=%E2%9C%93&sort=&provider=&q=ubuntu+desktop+14.04)などと入力して適当なBoxをみつけます。

今回は[Part2](/2014/06/08/packer-windows-vmware-iso-ova-build/)と同じ、Box-Cutterの[ ubuntu1404-desktop](https://vagrantcloud.com/box-cutter/ubuntu1404-desktop)を使います。

### Vagrantの開始

まずMinttyを起動します。
`Vagrant Cloud`からBoxを取得して、`vagrant up`すると、VirtualBoxのGUIが起動します。

``` bash
$ mkdir -p ~/vagrant_apps
$ cd !$
$ vagrant init box-cutter/ubuntu1404-desktop
$ vagrant up
```

`vagrant ssh`でなく、MinttyからSSH接続できるようにします。

``` bash
$ vagrant ssh-config --host desktop >> ~/.ssh/config
$ ssh desktop
Welcome to Ubuntu 14.04 LTS (GNU/Linux 3.13.0-24-generic x86_64)

 * Documentation:  https://help.ubuntu.com/

Last login: Sun Jun  8 14:52:15 2014 from 10.0.2.2
vagrant@ubuntu1404-desktop:~$
```

### VBoxManage.exeでディスクサイズの拡張
[Vagrant VMのディスクサイズを後から拡張する方法](http://blog.dakatsuka.jp/2014/04/24/vagrant-hdd-resize.html)を参考に、ディスクの拡張作業を行います。

まずVagrantを停止します。

``` bash
$ vagrant halt
```

Minttyのシェルに、VirtualBoxの実行ファイルへPATHが通っていることを確認します。

``` bash ~/.bash_profile
export PATH=/c/Program\ Files/Oracle/VirtualBox/:$PATH
```

仮想ディスクがVMDK形式だとからサイズ変更ができないため、VDI形式に変換します。
``` bash
$ cd ~/VirtualBox\ VMs/ubuntu1404-desktop/
$ VBoxManage.exe clonehd ubuntu1404-desktop-disk1.vmdk ubuntu1404-desktop-disk1.vdi --format VDI
0%...10%...20%...30%...40%...50%...60%...70%...80%...90%...100%
Clone hard disk created in format 'VDI'. UUID: 5e2ef773-98d7-43f1-8ffe-6e37273d17ce
```

とりあえず、40GBにディスクサイズを拡張します。

``` bash
$ VBoxManage modifyhd ubuntu1404-desktop-disk1.vdi --resize 40960
0%...10%...20%...30%...40%...50%...60%...70%...80%...90%...100%
```

### 仮想ディスクをVDIに交換

VirtualBoxマネージャから、ストレージを変換したVDI形式の仮想ディスクに交換します。

* ストレージ -> コントローラー: IDE Controller -> ubuntu1404-desktop-disk1.vmdk -> 割り当てを除去
* ストレージ -> コントローラー: IDE Controller -> ハードディスクの追加アイコン -> 既存のディスクを選択 -> ubuntu1404-desktop-disk1.vdi

古いVMDKの仮想ディスクを削除します。

``` bash
$ rm ~/VirtualBox\ VMs/ubuntu1404-desktop/ubuntu1404-desktop-disk1.vmdk
```

### 仮想マシンのパーティションテーブルを変更
仮想マシンを起動後、SSH接続をして確認します。

``` bash
$ cd ~/vagrant_apps/ubuntu1404-desktop/
$ vagrant up
$ ssh desktop
```
パーティションテーブルの変更作業をします。まずfdiskで`/dev/sda`の容量が増えている事を確認します。

``` bash
$ sudo fdisk -l
Disk /dev/sda: 42.9 GB, 42949672960 bytes
255 heads, 63 sectors/track, 5221 cylinders, total 83886080 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk identifier: 0x0005d213

   Device Boot      Start         End      Blocks   Id  System
/dev/sda1   *        2048      499711      248832   83  Linux
/dev/sda2          501758    20764671    10131457    5  Extended
/dev/sda5          501760    20764671    10131456   8e  Linux LVM
```

`/dev/sda2`と`/dev/sda5`のパーティションを一度削除して再定義します。

``` bash
$ sudo fdisk /dev/sda

Command (m for help): d
Partition number (1-5): 5

Command (m for help): d
Partition number (1-5): 2

Command (m for help): n
Partition type:
   p   primary (1 primary, 0 extended, 3 free)
   e   extended
Select (default p): e
Partition number (1-4, default 2):
Using default value 2
First sector (499712-83886079, default 499712):
Using default value 499712
Last sector, +sectors or +size{K,M,G} (499712-83886079, default 83886079):
Using default value 83886079

Command (m for help): n
Partition type:
   p   primary (1 primary, 1 extended, 2 free)
   l   logical (numbered from 5)
Select (default p): l
Adding logical partition 5
First sector (501760-83886079, default 501760):
Using default value 501760
Last sector, +sectors or +size{K,M,G} (501760-83886079, default 83886079):
Using default value 83886079
```

再定義できているか確認します。
``` bash
Command (m for help): p

Disk /dev/sda: 42.9 GB, 42949672960 bytes
255 heads, 63 sectors/track, 5221 cylinders, total 83886080 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk identifier: 0x0005d213

   Device Boot      Start         End      Blocks   Id  System
/dev/sda1   *        2048      499711      248832   83  Linux
/dev/sda2          499712    83886079    41693184    5  Extended
/dev/sda5          501760    83886079    41692160   83  Linux
```

`/dev/sda5`を`Linux LVM`に変更します。

``` bash
Command (m for help): t
Partition number (1-5): 5
Hex code (type L to list codes): 8e
Changed system type of partition 5 to 8e (Linux LVM)

Command (m for help): p

Disk /dev/sda: 42.9 GB, 42949672960 bytes
255 heads, 63 sectors/track, 5221 cylinders, total 83886080 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk identifier: 0x0005d213

   Device Boot      Start         End      Blocks   Id  System
/dev/sda1   *        2048      499711      248832   83  Linux
/dev/sda2          499712    83886079    41693184    5  Extended
/dev/sda5          501760    83886079    41692160   8e  Linux LVM
```

変更を保存してfdiskを終了します。
``` bash
Command (m for help): wq
The partition table has been altered!

Calling ioctl() to re-read partition table.

WARNING: Re-reading the partition table failed with error 16: Device or resource busy.
The kernel still uses the old table. The new table will be used at
the next reboot or after you run partprobe(8) or kpartx(8)
Syncing disks.
```

仮想マシンを再起動します。
``` bash
$ sudo reboot
```

### 仮想マシンのLVMの設定をする
pvresizeで物理ボリュームの`/dev/sda5`をリサイズします。
``` bash
$ sudo pvresize /dev/sda5
  Physical volume "/dev/sda5" changed
  1 physical volume(s) resized / 0 physical volume(s) not resized
```

pvscanで容量が増えていることを確認します。
``` bash
$ sudo pvscan
  PV /dev/sda5   VG ubuntu1404-desktop-vg   lvm2 [39.76 GiB / 30.10 GiB free]
  Total: 1 [39.76 GiB] / in use: 1 [39.76 GiB] / in no VG: 0 [0   ]
```

論理ボリューム名はlvscanで確認できます。
``` bash
$ sudo lvscan
  ACTIVE            '/dev/ubuntu1404-desktop-vg/root' [9.16 GiB] inherit
  ACTIVE            '/dev/ubuntu1404-desktop-vg/swap_1' [512.00 MiB] inherit
```

論理ボリュームをリサイズします。
``` bash
$ sudo lvresize -l +100%FREE /dev/ubuntu1404-desktop-vg/root
  Extending logical volume root to 39.26 GiB
  Logical volume root successfully resized
```

再度lvscanで容量が増えているか確認します。
``` bash
$ sudo lvscan
  ACTIVE            '/dev/ubuntu1404-desktop-vg/root' [39.26 GiB] inherit
  ACTIVE            '/dev/ubuntu1404-desktop-vg/swap_1' [512.00 MiB] inherit
```

resize2fsを使ってファイルシステムをリサイズします。
``` bash
$ sudo resize2fs /dev/ubuntu1404-desktop-vg/root
resize2fs 1.42.9 (4-Feb-2014)
Filesystem at /dev/ubuntu1404-desktop-vg/root is mounted on /; on-line resizing required
old_desc_blocks = 1, new_desc_blocks = 3
The filesystem on /dev/ubuntu1404-desktop-vg/root is now 10291200 blocks long.
```

最後にdfで確認してます。仮想マシンのディスクサイズが拡張されました!
```
$ df -h
Filesystem                                Size  Used Avail Use% Mounted on
/dev/mapper/ubuntu1404--desktop--vg-root   39G  3.2G   34G   9% /
none                                      4.0K     0  4.0K   0% /sys/fs/cgroup
udev                                      487M  4.0K  487M   1% /dev
tmpfs                                     100M  816K   99M   1% /run
none                                      5.0M     0  5.0M   0% /run/lock
none                                      498M   76K  497M   1% /run/shm
none                                      100M   48K  100M   1% /run/user
/dev/sda1                                 236M   37M  188M  17% /boot
```

### まとめ

VagrantからVirtualBox上にUbuntu14.04のデスクトップ環境と、PackerからOVAを作成するためにディスクサイズの拡張をしました。

次回は、この仮想マシン上にNestedの`VMware Pleyer`をインストールして、Packerの開発環境を構築します。

PackerのESXi5上の`remote build`はISOのアップロードを毎回行うバグがあり、直したいなと思っていたところ[Vmware ESXi remote builder always uploads iso to ESXi host](https://github.com/mitchellh/packer/issues/1244)でPRがありました。あとでビルドして試したいです。


