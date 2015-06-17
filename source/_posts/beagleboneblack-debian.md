title: 'BeagleBone Blackのファームウェアを入れ直す - Part1: Debian 7.5とWindows7作業環境'
date: 2015-01-27 03:22:10
tags:
 - Debian
 - BeagleBoneBlack
 - Cygwin
 - apt-cyg
description: 私のBeagleBone BlackのリビジョンはA5Bと古いのでeMMCが2GBしかありません。現在のファームウェアはUbuntu 13.04を使っているのですが、別の環境でSDカードに焼いた14.04が起動してくれません。一度ファームウェアをオフィシャルのDebianに戻してビルド環境を用意しようと思います。
---

私のBeagleBone BlackのリビジョンはA5Bと古いのでeMMCが2GBしかありません。現在のファームウェアはUbuntu 13.04を使っているのですが、別の環境でSDカードに焼いた14.04が起動してくれません。一度ファームウェアをオフィシャルのDebianに戻してビルド環境を用意しようと思います。

<!-- more -->

## Cygwinとapt-cygの用意

今回は作業環境にWindows7を使います。まずCygwinのセットアップのため[setup-x86_64.exe](https://cygwin.com/setup-x86_64.exe)をダウンロードしてインストールします。パッケージ管理ツールのapt-cygに必要なパッケージをインストールしておきます。

* Devel/git-svn
* Base/gawk
* Utils/bzip2
* Utils/tar
* Web/wget

Cygwin64 Terminalを管理者として起動してapt-cygをインストールします。

``` bash
$ svn --force export http://apt-cyg.googlecode.com/svn/trunk/ /bin/
A    /bin
A    /bin/LICENSE
A    /bin/apt-cyg
A    /bin/README.md
リビジョン 34 をエクスポートしました。
$ chmod +x /bin/apt-cyg
```

バージョンを確認します。

``` bash
$ cygcheck -c cygwin
Cygwin Package Information
Package              Version        Status
cygwin               1.7.28-2       OK
$ apt-cyg --version
apt-cyg version 0.59
Written by Stephen Jungels

Copyright (c) 2005-9 Stephen Jungels.  Released under the GPL.
```

apt-cygを使いunxzコマンドをインストールします。xz形式のイメージファイルの解凍に使います。

``` bash
$ apt-cyg install xz-utils
```

## Debian 7.5のインストール

### Windows7にUSBドライバをインストールする

WindowsからUSB経由でBBBにSSH接続するため、[Getting Started](http://beagleboard.org/getting-started)にある[64-bit版ドライバ](http://beagleboard.org/static/Drivers/Windows/BONE_D64.exe)のインストーラーをダウンロードして実行します。

![device-driver-installed.png](/2015/01//27/beagleboneblack-debian/device-driver-installed.png)


### microSDカードをフォーマットする

Debianのイメージを焼くため余っていた16GBのmicroSDカードを用意しました。[SD Formatter](https://www.sdcard.org/jp/downloads/formatter_4/)を使いmicroSDカードをフォーマットし直します。論理サイズ調整をONにしてクイックフォーマットします。

![sdformatter.png](/2015/01//27/beagleboneblack-debian/sdformatter.png)

フォーマットが終了しました。

![sdformatter.png](/2015/01//27/beagleboneblack-debian/sdformatter-finish.png)

### microSDカードにイメージを焼く

Cygwinを使いDebianのイメージをSDカードに焼きます。Cygwin64 TerminalからWindows7のデバイスファイルを確認します。16GBのmicroSDカードは`/dev/sdc`になります。

``` bash
$ cat /proc/partitions
major minor  #blocks  name

    8     0 488384323 sda
    8     1     40131 sda1
    8     2  17598464 sda2
    8     3 470743040 sda3
    8    16 102130688 sdb
    8    17   8388608 sdb1
    8    18  93740032 sdb2
    8    32  15351808 sdc
    8    33  15347712 sdc1
```

Debianのファームウェアはオフィシャルの[Latest Firmware Images](http://beagleboard.org/latest-images)にある、[Debian (BeagleBone Black - 2GB eMMC) 2014-05-14](http://debian.beagleboard.org/images/BBB-eMMC-flasher-debian-7.5-2014-05-14-2gb.img.xz)を使います。ダウンロードしてチェックサムを確認します。

``` bash
$ wget http://debian.beagleboard.org/images/BBB-eMMC-flasher-debian-7.5-2014-05-14-2gb.img.xz
$ md5sum BBB-eMMC-flasher-debian-7.5-2014-05-14-2gb.img.xz
74615fb680af8f252c034d3807c9b4ae *BBB-eMMC-flasher-debian-7.5-2014-05-14-2gb.img.xz
```

イメージを解凍してmicroSDカードに焼きます。

``` bash
$ unxz BBB-eMMC-flasher-debian-7.5-2014-05-14-2gb.img.xz
$ dd if=./BBB-eMMC-flasher-debian-7.5-2014-05-14-2gb.img of=/dev/sdc
3481600+0 レコード入力
3481600+0 レコード出力
1782579200 バイト (1.8 GB) コピーされました、 333.563 秒、 5.3 MB/秒
```

### eMMCにフラッシュする

以下の手順で、microSDカードからBBBのeMMCにDebianをフラッシュします。

* microSDカードをBBBのスロットに挿す
* microSDカードスロットの上にあるBootボタンを押しながら、USBをWindowsと接続してSDカードから起動する
* LEDが点滅し始めたら、Bootボタンを放す
* eMMCへのフラッシュには、20分くらいかかる
* フラッシュに成功すると、LEDが消灯して終了する
* Windowsから、USBケーブルを抜く
* BBBから、microSDカードを抜く

WindowsにUSBケーブルで接続してBBBの電源を入れます。Puttyを起動します。デフォルト接続設定は以下です。

* IPアドレス: 192.168.7.2
* username: debian
* password: temppwd

![debian-emmc.png](/2015/01//27/beagleboneblack-debian/debian-emmc.png)


PuttyからSSH接続できました。OSのバージョンを確認しておきます。

``` bash
$ cat /etc/debian_version
7.5
$ uname -a
Linux beaglebone 3.8.13-bone50 #1 SMP Tue May 13 13:24:52 UTC 2014 armv7l GNU/Linux
```


## Windows7のネットワーク設定

### USBネットワークの名前

BBBからUSBケーブルで接続しているWindowsのネットワークを使い、インターネットに接続できるようにします。わかりやすいように、USBネットワークの名前を変更しておきます。

* コントロールパネル > ネットワークとインターネット > ネットワークと共有センター > アダプターの設定の変更 > Linux USB Ethernet/RNDIS Gadgetのネットワークを選択
* 名前: BeagleBoneBlack


### ワイヤレスネットワークの共有設定

BBBと共有したネットワークの設定を変更します。今回はWindows7のワイヤレスネットワークを選択して共有タブを表示します。

* コントロールパネル > ネットワークとインターネット > ネットワークに接続 > 接続するネットワークを右クリック > 状態 > プロパティ > 共有タブ

以下のように、BeagleBone BlackのUSBネットワークからインターネットへの接続を許可します。

* ネットワークのほかのユーザーに、このコンピューターのインターネット接続をとおして接続を許可する: チェック
* ネットワークのほかのユーザーに、共有インターネット接続の制御や無効化を許可する: チェック外す
* ホームネットワーク接続: BeagleBoneBlackを選択 

![wifi-network-share.png](/2015/01//27/beagleboneblack-debian/wi-fi-network-share.png)

また、このワイヤレスネットワークは無線LANルーターからDHCPでIPアドレスとDNSサーバーを取得しています。

![wifi-dhcp.png](/2015/01//27/beagleboneblack-debian/wi-fi-dhcp.png)

### USBネットワーク

BBBのUSBネットワーク設定画面を以下の手順で開きます。

* コントロールパネル > ネットワークとインターネット > ネットワークの状態とタスクの表示 > アダプターの設定の変更 > Linux USB Ethernet/RNDIS Gadgetのネットワークを選択 

TCP/IPv4の設定画面を開きます。

* プロパティボタン > インターネット プロトコル バージョン 4 (TCP/IPv4) をダブルクリップ

以下のように、IPアドレスとDNSサーバーの情報を自動的に取得するようにします。

![bbb-network-dhcp.png](/2015/01//27/beagleboneblack-debian/bbb-network-dhcp.png)

## Debian 7.5のネットワーク設定

### デフォルトゲートウェイの追加

BBBとWindowsをUSBケーブルでつなぎ、PuttyからSSH接続します。routeコマンドでデフォルトゲートウエイの設定します。

``` bash
$ sudo route add default gw 192.168.7.1
```

routeコマンドで設定を確認します。

``` bash
$ route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         192.168.7.1     0.0.0.0         UG    0      0        0 usb0
192.168.7.0     0.0.0.0         255.255.255.252 U     0      0        0 usb0
```

pingで名前解決ができるようになりました。

``` bash
$ ping -c 3 www.yahoo.co.jp
PING www.g.yahoo.co.jp (183.79.135.206) 56(84) bytes of data.
64 bytes from f1.top.vip.kks.yahoo.co.jp (183.79.135.206): icmp_req=1 ttl=53 time=19.4 ms
...
```

### 再起動後も設定を有効にする

再起動後もデフォルトゲートウェイが有効になるように設定を追加します。[Windows 7 Internet Sharing for BeagleBone Black](http://lanceme.blogspot.jp/2013/06/windows-7-internet-sharing-for.html)のコメントにあるDebian用の書き方を参考にします。udhcpdのすぐ下にデフォルトゲートウェイの設定を追加します。

``` bash:/opt/scripts/boot/am335x_evm.sh
/sbin/ifconfig usb0 192.168.7.2 netmask 255.255.255.252
/usr/sbin/udhcpd -S /etc/udhcpd.conf
/sbin/route add default gw 192.168.7.1
```

BBBを再起動しても名前解決ができるようになりました。

``` bash
$ ping -c 3 www.yahoo.co.jp
PING www.g.yahoo.co.jp (183.79.27.149) 56(84) bytes of data.
64 bytes from f12.top.vip.kks.yahoo.co.jp (183.79.27.149): icmp_req=1 ttl=53 time=22.8 ms
...
```


