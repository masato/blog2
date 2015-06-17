title: "BeagleBone Blackのファームウェアを入れ直す - Part5: SDカードのUbuntu14.04.1とWindows7を接続する"
date: 2015-02-01 23:24:03
tags:
 - Ubuntu
 - BeagleBoneBlack
 - Linux-ARM
 - Windows7
description: Windows7環境のBeagleBone Blackは無線LANの設定をしてインターネットに接続できるようにしました。まだUSB Gadgetを使ったUSB-Ethernet接続ができません。g-etherカーネルモジュールのロードに問題があるようです。とりあえずカーネルを最新にするとg-etherモジュールが正常にロードされました。
---

Windows7環境のBeagleBone Blackは無線LANの設定をしてインターネットに接続できるようにしました。まだUSB Gadgetを使ったUSB-Ethernet接続ができません。`g-ether`カーネルモジュールのロードに問題があるようです。とりあえずカーネルを最新にすると`g-ether`モジュールが正常にロードされました。


<!-- more -->

## g-etherのインストールは失敗

PuttyからUSB-シリアル接続をしてコンソールを開きます。[Beaglebone Black (BBB) Cross Compile](http://dumb-looks-free.blogspot.jp/2014/05/beaglebone-black-bbb-cross-compile_28.html)を参考にして、USB Gadgetの`g-ether`モジュールをセットアップします。最初にudhcpdをインストールします。

``` bash
$ sudo apt-get install udhcpd
```

セットアップ用のスクリプトをダウンロードします。

``` bash
$ wget https://raw.githubusercontent.com/RobertCNelson/tools/master/scripts/beaglebone-black-g-ether-load.sh
$ chmod +x beaglebone-black-g-ether-load.sh
```

セットアップスクリプトを実行しますが、`'g_multi': No such device`とエラーが表示され失敗してしまいます。

``` bash
$ sudo ./beaglebone-black-g-ether-load.sh
cpsw.0: 90:59:AF:58:32:D1
cpsw.1: 90:59:AF:58:32:D3
modprobe: ERROR: could not insert 'g_multi': No such device
Stopping very small Busybox based DHCP server: No /usr/sbin/udhcpd found running; none killed.
udhcpd.
Starting very small Busybox based DHCP server: Starting /usr/sbin/udhcpd...
udhcpd.
SIOCSIFADDR: No such device
usb0: ERROR while getting interface flags: No such device
SIOCSIFNETMASK: No such device
```

### カーネルモジュールがロードされない

beaglebone-black-g-ether-load.shの中で実行しているカーネルモジュールのロードをデバックしてみます。

``` bash
$ sudo modprobe g_ether -v
insmod /lib/modules/3.14.26-ti-r43/kernel/drivers/usb/gadget/g_ether.ko
modprobe: ERROR: could not insert 'g_ether': No such device
```

`g_ether`のデバイスドライバが見つからないようですが、findするとちゃんとあります。

``` bash
$ sudo find / -name 'g_ether.ko' -print
/lib/modules/3.14.26-ti-r43/kernel/drivers/usb/gadget/g_ether.ko
```

dmsgを見ると`g_ether`モジュールは`kernel tainted`のためロードされません。GPL汚染フラグが立っているようです。

``` bash
$ sudo dmesg |tail
[  309.403366] g_ether: module_layout: kernel tainted.
[  309.403425] Disabling lock debugging due to kernel taint
```

### 最新のカーネルをインストールする

`g_ether`を有効にするにはカーネルモジュールのコンフィグが必要です。[Install Latest Kernel Image](http://elinux.org/BeagleBoardUbuntu#Install_Latest_Kernel_Image)を参考にして、まずはカーネルを最新バージョンに更新してみます。現在のカーネルのバージョンを確認します。
 
``` bash
$ uname -a
Linux arm 3.14.26-ti-r43 #1 SMP PREEMPT Wed Dec 24 05:27:12 UTC 2014 armv7l armv7l armv7l GNU/Linux
```

最新のソースをpullします。

``` bash
$ cd /opt/scripts/tools
$ git pull
remote: Counting objects: 59, done.
remote: Compressing objects: 100% (43/43), done.
remote: Total 59 (delta 45), reused 30 (delta 16)
Unpacking objects: 100% (59/59), done.
From https://github.com/RobertCNelson/boot-scripts
   7068eda..e6fd16e  master     -> origin/master
Updating 7068eda..e6fd16e
Fast-forward
 boot/am335x_evm.sh                                            | 15 ++++--
 boot/beagle_x15.sh                                            | 20 ++-----
 boot/omap5_uevm.sh                                            | 22 +++-----
 mods/wheezy-systemd-poweroff.diff                             | 22 ++++++++
 tools/developers/update_bootloader.sh                         | 87 +++++++++++++++++--------------
 tools/eMMC/bbb-eMMC-flasher-eewiki-ext4.sh                    |  8 +--
 tools/eMMC/beaglebone-black-make-microSD-flasher-from-eMMC.sh |  8 +--
 tools/eMMC/init-eMMC-flasher-v3.sh                            |  9 ++--
 tools/update_kernel.sh                                        |  9 ++--
 9 files changed, 110 insertions(+), 90 deletions(-)
 create mode 100644 mods/wheezy-systemd-poweroff.diff
```

カーネルイメージをビルドして再起動します。

``` bash
$ sudo ./update_kernel.sh
...
Selecting previously unselected package ti-sgx-es8-modules-3.14.31-ti-r49.
(Reading database ... 22299 files and directories currently installed.)
Preparing to unpack .../ti-sgx-es8-modules-3.14.31-ti-r49_1trusty_armhf.deb ...
Unpacking ti-sgx-es8-modules-3.14.31-ti-r49 (1trusty) ...
Setting up ti-sgx-es8-modules-3.14.31-ti-r49 (1trusty) ...
update-initramfs: Generating /boot/initrd.img-3.14.31-ti-r49
$ sudo reboot
```

再起動するとカーネルが更新されます。

``` bash
$ uname -a
$ Linux arm 3.14.31-ti-r49 #1 SMP PREEMPT Sat Jan 31 14:17:42 UTC 2015 armv7l armv7l armv7l GNU/Linux
```

`g_ether`モジュールがロードされるようになりました。

``` bash
$ lsmod
Module                  Size  Used by
ctr                     3277  1
ccm                     7172  1
usb_f_ecm               7901  1
g_ether                 1798  0
usb_f_rndis            17711  2 g_ether
u_ether                 9524  3 usb_f_ecm,usb_f_rndis,g_ether
libcomposite           38715  3 usb_f_ecm,usb_f_rndis,g_ether
arc4                    1602  2
rt2800usb              15756  0
rt2800lib              70568  1 rt2800usb
rt2x00usb               8125  1 rt2800usb
rt2x00lib              35077  3 rt2x00usb,rt2800lib,rt2800usb
mac80211              439647  3 rt2x00lib,rt2x00usb,rt2800lib
cfg80211              380240  2 mac80211,rt2x00lib
crc_ccitt               1110  1 rt2800lib
rfkill                 14659  2 cfg80211
musb_dsps               8369  0
musb_hdrc              76236  1 musb_dsps
pvrsrvkm              183463  0
c_can_platform          5927  0
c_can                   9400  1 c_can_platform
can_dev                 7430  1 c_can
musb_am335x             1075  0
```

`g_ether`モジュールがロードされるとusb0のネットワークが作成されました。

``` bash
$ ip addr show usb0
5: usb0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether ee:2c:31:30:53:23 brd ff:ff:ff:ff:ff:ff
    inet 192.168.7.2/30 brd 192.168.7.3 scope global usb0
       valid_lft forever preferred_lft forever
    inet6 fe80::ec2c:31ff:fe30:5323/64 scope link
       valid_lft forever preferred_lft forever
```

## USB-Ethernet接続の確認

### ネットワークの共有設定

BeagleBone BlackとWindows7とUSB接続すると、Linux USB Ethernet/RNDIS Gadgetのネットワークが作成されます。プロパティを開きTCP/IPv4の設定をDHCPに変更します。

![beagleboneblack-tcpip.png](/2015/02/01/beagleboneblack-ubuntu14-04-windows7/beagleboneblack-tcpip.png)

インターネット接続を共有するため、ワイヤレスネットワーク接続のプロパティを開きます。共有タブからUSB Gadgetとネットワーク共有を許可します。

![wifi-share.png](/2015/02/01/beagleboneblack-ubuntu14-04-windows7/wifi-share.png)

### SSH接続

PuttyからBeagleBone Blackの192.168.7.2へSSH接続します。

``` bash
Using username "ubuntu".
Ubuntu 14.04.1 LTS

rcn-ee.net console Ubuntu Image 2015-01-06

Support/FAQ: http://elinux.org/BeagleBoardUbuntu

default username:password is [ubuntu:temppwd]

ubuntu@192.168.7.2's password:
Welcome to Ubuntu 14.04.1 LTS (GNU/Linux 3.14.31-ti-r49 armv7l)
```

デフォルトゲートウェイの設定をして、ルーティングを確認します。

``` bash
$ sudo route add default gw 192.168.7.1
$ route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         192.168.7.1     0.0.0.0         UG    0      0        0 usb0
192.168.7.0     0.0.0.0         255.255.255.252 U     0      0        0 usb0
```

BeagleBone BlackからWindows7のUSB-Ethernet接続を経由して、名前解決とインターネットに接続ができるようになりました。

``` bash
$ ping -c 3 www.yahoo.co.jp
PING www.g.yahoo.co.jp (183.79.197.250) 56(84) bytes of data.
64 bytes from 183.79.197.250: icmp_seq=1 ttl=53 time=33.1 ms
```
