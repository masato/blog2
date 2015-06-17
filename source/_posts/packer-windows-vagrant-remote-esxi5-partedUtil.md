title: 'Packerを使いWindows上でOVAを作成する - Part7: Nested ESXi5 on VirtualBox パーティション作成'
date: 2014-06-14 18:16:15
tags:
 - Packer
 - PackerOVA
 - Go
 - VirtualBox
 - Windows
 - ESXi5
 - partedUtil
 - NestedVirtualization
description: Part6でPackerからリモートのESXi5を使ったビルドに失敗してしまったのですが、あきらめずに少しずつ調べながら理解していきます。どうもESXi5上にディスク領域が足りないので、あたらしいデバイスを追加してデータストアを作成する必要がありそうです。Using vmkfstools to Manage VMFS Datastoresを参考にしながら、ESXi5の操作を勉強していきます。
---

[PackerでOVA作成](http://masato.github.io/tags/PackerOVA/)シリーズです。
[Part6](/2014/06/13/packer-windows-vagrant-remote-esxi5-build/)でPackerからリモートのESXi5を使ったビルドに失敗してしまったのですが、あきらめずに少しずつ調べながら理解していきます。
どうもESXi5上にディスク領域が足りないので、あたらしいデバイスを追加してデータストアを作成する必要がありそうです。
[Using vmkfstools to Manage VMFS Datastores](http://buildvirtual.net/using-vmkfstools-to-manage-vmfs-datastores/)を参考にしながら、ESXi5の操作を勉強していきます。

<!-- more -->

### ESXi5にログインして確認

昨日はISOファイルのアップロードが終了していないところまで判明しましたが、途中で力尽きてしまいました。

ESXi5にSSHログインして

``` bash
$ ssh root@192.168.56.101
$ ls -al /vmfs/volumes/datastore1/packer_cache
total 508936
drwxr-xr-x    1 root     root           420 Jun 14 08:11 .
drwxr-xr-t    1 root     root          1400 Jun 14 08:24 ..
-rw-r--r--    1 root     root     522715136 Jun 14 08:24 616e30c4df43460f8b93c3b5a9efb08868eb9b4778c5f5014607a37095b91524.iso
```

アップロードに失敗したISOを削除します。

```
$ rm /vmfs/volumes/datastore1/packer_cache/616e30c4df43460f8b93c3b5a9efb08868eb9b4778c5f5014607a37095b91524.iso
```

ESXi5のディスクの空き領域を見ると、

``` bash
$ df -h
Filesystem   Size   Used Available Use% Mounted on
VMFS-5     512.0M  16.0M    496.0M   3% /vmfs/volumes/datastore1
vfat         4.0G   3.2M      4.0G   0% /vmfs/volumes/539bdf88-79d4ec57-39ff-080027796f03
vfat       249.7M 157.8M     91.9M  63% /vmfs/volumes/2528a071-f798a567-2aae-10c228e9dbc1
vfat       249.7M   8.0K    249.7M   0% /vmfs/volumes/e575b17d-ca145a97-a11c-7309b6886740
vfat       285.8M 192.6M     93.2M  67% /vmfs/volumes/539bdf7d-217c0e72-3c01-080027796f03
```

522715136Bytesは、498.5MBなので、ISOをアップロードしていたdatastore1の領域が足りないようです。

### VirtualBoxからディスクの追加

ESXi5の仮想マシンを停止して、VirtualBoxマネージャーからディスクを追加します。
[Part5](/2014/06/12/packer-windows-vagrant-nested-esxi5/)で8GBのディスクを作成した時と同じ方法です。


### partedUtilで追加したデバイスの確認

esxcliコマンドで追加したディスクのパスを確認すると、SATAのディスクが2つみつかります。

``` bash
$ esxcli storage core path list
...
   Device: t10.ATA_____VBOX_HARDDISK___________________________VB3708b3ff2D6ab73e63_
...
   Device: t10.ATA_____VBOX_HARDDISK___________________________VB375d482d2D7e948af6_
...
```
partedUtilコマンドを使い、パーティションが作成されていないディスクを確認します。

既存のディスクの場合、すでにパーティションが存在しています。

``` bash
$/sbin/partedUtil "getptbl" "/vmfs/devices/disks/t10.ATA_____VBOX_HARDDISK___________________________VB375d482d2D7e948af6_"
gpt
1044 255 63 16777216
1 64 8191 C12A7328F81F11D2BA4B00A0C93EC93B systemPartition 128
5 8224 520191 EBD0A0A2B9E5443387C068B6B72699C7 linuxNative 0
6 520224 1032191 EBD0A0A2B9E5443387C068B6B72699C7 linuxNative 0
7 1032224 1257471 9D27538040AD11DBBF97000C2911D1B8 vmkDiagnostic 0
8 1257504 1843199 EBD0A0A2B9E5443387C068B6B72699C7 linuxNative 0
9 1843200 7086079 9D27538040AD11DBBF97000C2911D1B8 vmkDiagnostic 0
2 7086080 15472639 EBD0A0A2B9E5443387C068B6B72699C7 linuxNative 0
3 15472640 16777182 AA31E02A400F11DB9590000C2911D1B8 vmfs 0
```

追加したディスクは、以下の方だとわかります。
`t10.ATA_____VBOX_HARDDISK___________________________VB3708b3ff2D6ab73e63_`

``` bash
$ /sbin/partedUtil "getptbl" "/vmfs/devices/disks/t10.ATA_____VBOX_HARDDISK___________________________VB3708b3ff2D6ab73e63_"
unknown
1044 255 63 16777216
```

### エンドセクターの計算

出力結果の意味は、それぞれ以下になります。

* 1044: シリンダー数
* 255: ヘッダ数
* 63: トラック毎のセクター数
* 16777216: セクター数

エンドセクターの計算式は、(C x H x S -1)なので、16771859になります。

``` python
$ python
>>> (1044 * 255 * 63 ) -1
16771859
```

### パーティションタイプの確認

作成するパーティションのタイプを確認します。
VMFSのGUIDは、`AA31E02A400F11DB9590000C2911D1B8`です。

```
$ /sbin/partedUtil showGuids
 Partition Type       GUID
 vmfs                 AA31E02A400F11DB9590000C2911D1B8
 vmkDiagnostic        9D27538040AD11DBBF97000C2911D1B8
 vsan                 381CFCCC728811E092EE000C2911D0B2
 VMware Reserved      9198EFFC31C011DB8F78000C2911D1B8
 Basic Data           EBD0A0A2B9E5443387C068B6B72699C7
 Linux Swap           0657FD6DA4AB43C484E50933C84B4F4F
 Linux Lvm            E6D6D379F50744C2A23C238F2A3DF928
 Linux Raid           A19D880F05FC4D3BA006743F0F84911E
 Efi System           C12A7328F81F11D2BA4B00A0C93EC93B
 Microsoft Reserved   E3C9E3160B5C4DB8817DF92DF00215AE
 Unused Entry         00000000000000000000000000000000
```

### パーティションの作成

追加したデバイスにパーティションを作成します。

``` bash
$ partedUtil setptbl /vmfs/devices/disks/t10.ATA_____VBOX_HARDDISK___________________________VB3708b3ff2D6ab73e63_ gpt "1 2048 16771859 AA31E02A400F11DB9590000C2911D1B8 0"
gpt
0 0 0 0
1 2048 16771859 AA31E02A400F11DB9590000C2911D1B8 0
```

作成したパーティションを確認します。ちゃんとできました。

``` bash
$ /sbin/partedUtil "getptbl" "/vmfs/devices/disks/t10.ATA_____VBOX_HARDDISK_______________________
____VB3708b3ff2D6ab73e63_"
gpt
1044 255 63 16777216
1 2048 16771859 AA31E02A400F11DB9590000C2911D1B8 vmfs 0
```

### VMFSのデータストアの作成
ディスクを確認すると、パーティションができています。

``` bash
$ ls /vmfs/devices/disks/
t10.ATA_____VBOX_HARDDISK___________________________VB3708b3ff2D6ab73e63_
t10.ATA_____VBOX_HARDDISK___________________________VB3708b3ff2D6ab73e63_:1
t10.ATA_____VBOX_HARDDISK___________________________VB375d482d2D7e948af6_
t10.ATA_____VBOX_HARDDISK___________________________VB375d482d2D7e948af6_:1
t10.ATA_____VBOX_HARDDISK___________________________VB375d482d2D7e948af6_:2
t10.ATA_____VBOX_HARDDISK___________________________VB375d482d2D7e948af6_:3
t10.ATA_____VBOX_HARDDISK___________________________VB375d482d2D7e948af6_:5
t10.ATA_____VBOX_HARDDISK___________________________VB375d482d2D7e948af6_:6
t10.ATA_____VBOX_HARDDISK___________________________VB375d482d2D7e948af6_:7
t10.ATA_____VBOX_HARDDISK___________________________VB375d482d2D7e948af6_:8
t10.ATA_____VBOX_HARDDISK___________________________VB375d482d2D7e948af6_:9
```

vmkftoolsを使い、新しいデータストアを作成します。`NewVol`というラベルをつけました。

``` bash
$ vmkfstools --createfs vmfs5 --blocksize 1m -S NewVol /vmfs/devices/disks/t10.ATA_____VBOX_HARDDI
SK___________________________VB3708b3ff2D6ab73e63_:1
create fs deviceName:'/vmfs/devices/disks/t10.ATA_____VBOX_HARDDISK___________________________VB3708b3ff2D6ab73e63_:1', fsShortName:'vmfs5', fsName:'NewVol'
deviceFullPath:/dev/disks/t10.ATA_____VBOX_HARDDISK___________________________VB3708b3ff2D6ab73e63_:1 deviceFile:t10.ATA_____VBOX_HARDDISK___________________________VB3708b3ff2D6ab73e63_:1
Checking if remote hosts are using this device as a valid file system. This may take a few seconds...
Creating vmfs5 file system on "t10.ATA_____VBOX_HARDDISK___________________________VB3708b3ff2D6ab73e63_:1" with blockSize 1048576 and volume label "NewVol".
Successfully created new volume: 539c30a4-ec993a20-0f81-080027796f03
```

作成したボリュームの確認します。

``` bash
$ ls -al /vmfs/volumes/
total 3076
drwxr-xr-x    1 root     root           512 Jun 14 11:24 .
drwxr-xr-x    1 root     root           512 Jun 14 10:39 ..
drwxr-xr-x    1 root     root             8 Jan  1  1970 2528a071-f798a567-2aae-10c228e9dbc1
drwxr-xr-x    1 root     root             8 Jan  1  1970 539bdf7d-217c0e72-3c01-080027796f03
drwxr-xr-t    1 root     root          1400 Jun 14 09:11 539bdf84-f652f3f9-ee0a-080027796f03
drwxr-xr-x    1 root     root             8 Jan  1  1970 539bdf88-79d4ec57-39ff-080027796f03
drwxr-xr-t    1 root     root          1260 Jun 14 11:23 539c30a4-ec993a20-0f81-080027796f03
lrwxr-xr-x    1 root     root            35 Jun 14 11:24 NewVol -> 539c30a4-ec993a20-0f81-080027796f03
lrwxr-xr-x    1 root     root            35 Jun 14 11:24 datastore1 -> 539bdf84-f652f3f9-ee0a-080027796f03
drwxr-xr-x    1 root     root             8 Jan  1  1970 e575b17d-ca145a97-a11c-7309b6886740
```

dfコマンドで見ると、6.9Gの空き領域があります。

``` bash
$ df -h
Filesystem   Size   Used Available Use% Mounted on
VMFS-5     512.0M  16.0M    496.0M   3% /vmfs/volumes/datastore1
VMFS-5       7.8G 870.0M      6.9G  11% /vmfs/volumes/NewVol
vfat         4.0G   4.3M      4.0G   0% /vmfs/volumes/539bdf88-79d4ec57-39ff-080027796f03
vfat       249.7M 157.8M     91.9M  63% /vmfs/volumes/2528a071-f798a567-2aae-10c228e9dbc1
vfat       249.7M   8.0K    249.7M   0% /vmfs/volumes/e575b17d-ca145a97-a11c-7309b6886740
vfat       285.8M 192.6M     93.2M  67% /vmfs/volumes/539bdf7d-217c0e72-3c01-080027796f03
```

### まとめ

6.9Gの空き領域のデータストアができました。
次回は、新しいデータストアをPackerの`remote_datastore`に指定して、OVA作成を行ってみます。
