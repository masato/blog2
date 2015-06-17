title: "Snappy Ubuntu Core - Part2: OVAを作成してIDCFクラウドにデプロイする"
date: 2014-12-15 21:41:23
tags:
 - IDCFクラウド
 - IDCFオブジェクトストレージ
 - ovftool
 - OVA
 - VMX
 - Packer
 - SnappyUbuntuCore
description: Snappy Ubuntu Coreのqcow2ファイルをVMDKファイルに変換してOSXのVirtualBoxへデプロイは成功しましたが、VirtualBoxからexportしたOVAファイルはIDCFクラウドにテンプレートとして登録できませんでした。今回はVMXファイルを手動で作成してovftoolを使いOVAファイルを作成してみます。
---

Snappy Ubuntu Coreのqcow2ファイルをVMDKファイルに変換して[OSXのVirtualBox](/2014/12/12/snappy-ubuntu-core-virtualbox/)へデプロイは成功しましたが、VirtualBoxからexportしたOVAファイルはIDCFクラウドにテンプレートとして登録できませんでした。今回はVMXファイルを手動で作成してovftoolを使いOVAファイルを作成してみます。

<!-- more -->

### ovftoolのインストール

最初にVMwareアカウントを作成します。ブラウザで[OVF Tool Downloadページ](https://my.vmware.com/web/vmware/details?downloadGroup=OVFTOOL400&productId=353)にアクセスしてVMware OVF Tool 4.0.0 for Linux 64 bitをダウンロードします。ダウンロードした`VMware-ovftool-4.0.0-2301625-lin.x86_64.bundle`を作業マシンにSCP転送します。

ovftoolをインストールします。

``` bash
$ sudo /bin/sh ~/VMware-ovftool-4.0.0-2301625-lin.x86_64.bundle
...
Installing VMware OVF Tool component for Linux 4.0.0
Copying files...
Configuring...
Installation was successful.
```

### VMXを作成する

Packerの[step_create_vmx.go](https://github.com/mitchellh/packer/blob/master/builder/vmware/iso/step_create_vmx.go)にあるVMXのテンプレートを参考にして、VMXファイルを作成します。

``` bash ~/ova/UbuntuCore.vmx
.encoding = "UTF-8"
bios.bootOrder = "hdd,CDROM"
checkpoint.vmState = ""
cleanShutdown = "TRUE"
config.version = "8"
displayName = "Ubuntu Core"
ehci.pciSlotNumber = "34"
ehci.present = "TRUE"
ethernet0.addressType = "generated"
ethernet0.bsdName = "en0"
ethernet0.connectionType = "nat"
ethernet0.displayName = "Ethernet"
ethernet0.linkStatePropagation.enable = "FALSE"
ethernet0.pciSlotNumber = "33"
ethernet0.present = "TRUE"
ethernet0.virtualDev = "e1000"
ethernet0.wakeOnPcktRcv = "FALSE"
extendedConfigFile = "Ubuntu Core.vmxf"
floppy0.present = "FALSE"
guestOS = "ubuntu-64"
gui.fullScreenAtPowerOn = "FALSE"
gui.viewModeAtPowerOn = "windowed"
hgfs.linkRootShare = "TRUE"
hgfs.mapRootShare = "TRUE"
isolation.tools.hgfs.disable = "FALSE"
memsize = "512"
nvram = "Ubuntu Core.nvram"
pciBridge0.pciSlotNumber = "17"
pciBridge0.present = "TRUE"
pciBridge4.functions = "8"
pciBridge4.pciSlotNumber = "21"
pciBridge4.present = "TRUE"
pciBridge4.virtualDev = "pcieRootPort"
pciBridge5.functions = "8"
pciBridge5.pciSlotNumber = "22"
pciBridge5.present = "TRUE"
pciBridge5.virtualDev = "pcieRootPort"
pciBridge6.functions = "8"
pciBridge6.pciSlotNumber = "23"
pciBridge6.present = "TRUE"
pciBridge6.virtualDev = "pcieRootPort"
pciBridge7.functions = "8"
pciBridge7.pciSlotNumber = "24"
pciBridge7.present = "TRUE"
pciBridge7.virtualDev = "pcieRootPort"
powerType.powerOff = "soft"
powerType.powerOn = "soft"
powerType.reset = "soft"
powerType.suspend = "soft"
proxyApps.publishToHost = "FALSE"
replay.filename = ""
replay.supported = "FALSE"
scsi0.pciSlotNumber = "16"
scsi0.present = "TRUE"
scsi0.virtualDev = "lsilogic"
scsi0:0.fileName = "ubuntu-core-alpha-01.vmdk"
scsi0:0.present = "TRUE"
scsi0:0.redo = ""
sound.startConnected = "FALSE"
tools.syncTime = "TRUE"
tools.upgrade.policy = "upgradeAtPowerCycle"
usb.pciSlotNumber = "32"
usb.present = "FALSE"
virtualHW.productCompatibility = "hosted"
virtualHW.version = "10"
vmci0.id = "1861462627"
vmci0.pciSlotNumber = "35"
vmci0.present = "TRUE"
vmotion.checkpointFBSize = "65536000"
```

### VMXからOVAを作成する

OVAを作成するプロジェクトです。

``` bash
$ tree ~/ova
~/ova
|-- UbuntuCore.vmx
`-- ubuntu-core-alpha-01.vmdk
```

ovftoolを使いVMXファイルからOVAファイルを作成します。

``` bash 
$ ovftool UbuntuCore.vmx UbuntuCore.ova
Opening VMX source: UbuntuCore.vmx
Opening OVA target: UbuntuCore.ova
Writing OVA package: UbuntuCore.ova
Transfer Completed
Completed successfully
```

作成したOVAファイルはs3cmdを使い、IDCFオブジェクトストレージにputします。

``` bash
$ s3cmd mb s3://my-ova
$ s3cmd put --acl-public UbuntuCore.ova s3://my-ova
```

上記の場合のOVAダウンロードのURLは以下になります。

http://my-ova.ds.jp-east.idcfcloud.com/UbuntuCore.ova


### IDCFクラウドにテンプレートを作成する

IDCFクラウドのダッシュボードにログインして、テンプレート画面を表示します。上記URLをOVAのURLを指定してOVAテンプレートを作成します。テンプレート名、説明、URL、OSタイプ以外はすべてデフォルトです。

* テンプレート名: UbuntuCore
* 説明: UbuntuCore
* URL: http://my-ova.ds.jp-east.idcfcloud.com/UbuntuCore.ova
* ゾーン: tesla
* ハイパーバイザー: VMware
* OSタイプ: Ubuntu 12.04 (64-bit)
* フォーマット: OVA
* エクスポート: 有効
* パスワードリセット: 有効
* ダイナミックスケール: 有効
* ルートディスクコントローラ: scsi
* NICアダプタ: Vmxnet3
* キーボード: Japanese

### IDCFクラウドに仮想マシンを作成する

OVAテンプレートが作成できたら、新しい仮想マシンを作成します。

### 確認

Snappy Ubuntu CoreにSSH接続します。username/passwordは、それぞれubuntu/ubuntuです。

``` bash
$ ssh ubuntu@10.3.0.9
Welcome to Ubuntu Vivid Vervet (development branch) (GNU/Linux 3.16.0-25-generic x86_64)

 * Documentation:  https://help.ubuntu.com/

The programs included with the Ubuntu system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Ubuntu comes with ABSOLUTELY NO WARRANTY, to the extent permitted by
applicable law.

Welcome to the Ubuntu Core rolling development release.

 * See https://ubuntu.com/snappy

It's a brave new world here in snappy Ubuntu Core! This machine
does not use apt-get or deb packages. Please see 'snappy --help'
for app installation and transactional updates.

To run a command as administrator (user "root"), use "sudo <command>".
See "man sudo_root" for details.

ubuntu@localhost:~$
```

snappyコマンドも使えるようになりました。

``` bash
$ snappy versions
Part         Tag   Installed  Available  Fingerprint     Active
ubuntu-core  edge  140        141        184ad1e863e947  *
```
