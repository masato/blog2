title: 'BeagleBone BlackにUbuntuをインストール'
date: 2014-05-03 03:22:10
tags:
 - Ubuntu
 - BeagleBoneBlack
 - Linux-ARM
description: RasberryPiが未開封のままなのに、USBで給電とシリアル接続ができるBeagleBone Blackが欲しくなりました。どの販売店も売り切れ状態だったので、ちょっと高いですがヤフオクから落札しました。
---

[RasberryPi](http://www.raspberrypi.org/)が未開封のままなのに、USBで給電とシリアル接続ができる[BeagleBone Black](http://beagleboard.org/Products/BeagleBone+Black)が欲しくなりました。どの販売店も売り切れ状態だったので、ちょっと高いですがヤフオクから落札しました。

<!-- more -->

### microSDカードのフォーマット

Windows7のノートPCにCygwinをインストールして作業環境にします。2GB以上のmicroSDカードが必要なので、最近は16GBのmicroSDHCカードでも1,000円以下で購入できました。[SD Formatter](https://www.sdcard.org/jp/downloads/formatter_4/)を使い、オプション設定ボタンから、以下を選択してフォーマットします。

* 消去方法: クイックフォーマット
* 論理サイズ調整: ON

### Cygwinを使いUbuntuのイメージをコピー

`Cygwin64 Terminal`を管理者として実行します。16GBのmicroSDカードのデバイス名は/dev/sdcです。

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
    8    32  15482880 sdc
    8    33     98304 sdc1
```

BeagleBone BlackのリビジョンはA5Bです。 なぜか`Trusty(14.04)`のイメージだとeMMCへのフラッシュに失敗してしまうので`Sausy(13.10)`を使うことにします。[Embedded Linux Wiki](http://elinux.org/BeagleBoardUbuntu#eMMC:_BeagleBone_Black)のページに従ってeMMCフラッシュ用のイメージをmicroSDカードにコピーします。

``` bash
$ wget https://rcn-ee.net/deb/flasher/saucy/BBB-eMMC-flasher-ubuntu-13.10-2014-03-27-2gb.img.xz
$ unxz BBB-eMMC-flasher-ubuntu-13.10-2014-03-27-2gb.img.xz
$ dd if=./BBB-eMMC-flasher-ubuntu-13.10-2014-03-27-2gb.img of=/dev/sdc
3481600+0 レコード入力
3481600+0 レコード出力
1782579200 バイト (1.8 GB) コピーされました、 552.543 秒、 3.2 MB/秒
```

### eMMCにUbuntuをインストール

ちょっとわかりずらいですが、以下の手順で行います。SDカードスロットの、すぐ上にあるBootボタンを押しながら電源を入れるとmicroSDカードからBootします。eMMCへのフラッシュには、だいたい1時間くらいかかります。

* microSDカードをBeagleBone Blackに挿入する。
* Bootボタンを押しながら、電源供給用のUSBケーブルをノートPCのUSBポートに接続する。
* LEDが点滅し始めたら、Bootボタンを放す。
* 4つのLEDが点灯したら、microSDカードから、起動終了
* ノートPCから、USBケーブルを抜く。
* BeagleBone Blackから、microSDカードを抜く。

### Windows7からUSB経由でネットワーク接続する

Windowsドライバを[ダウンロード](http://beagleboard.org/static/beaglebone/latest/Drivers/Windows/BONE_D64.exe)します。セキュリティ警告が何度か出ますがインストールできます。PuttyからSSHで接続します。初期ユーザーとパスワードは以下です。

* user: ubuntu
* pass: temppwd

``` bash
$ cat /etc/lsb-release
DISTRIB_ID=Ubuntu
DISTRIB_RELEASE=13.10
DISTRIB_CODENAME=saucy
DISTRIB_DESCRIPTION="Ubuntu 13.10"
$ df -h
Filesystem      Size  Used Avail Use% Mounted on
/dev/mmcblk0p2  1.7G  392M  1.2G  25% /
none            4.0K     0  4.0K   0% /sys/fs/cgroup
udev            248M  4.0K  248M   1% /dev
tmpfs            50M  272K   50M   1% /run
none            5.0M     0  5.0M   0% /run/lock
none            249M     0  249M   0% /run/shm
none            100M     0  100M   0% /run/user
/dev/mmcblk0p1   96M   72M   25M  75% /boot/uboot
```

### 無線LANの設定

パッケージのダウンロードなどインターネット接続は無線LANを経由して行います。インタフェースの設定は、DHCPとWPAにしてからupします。pingでインターネットにつながるか確認します。

``` bash
$ sudo cp /etc/network/interfaces /etc/network/interfaces-backup
$ sudo vi /etc/network/interfaces
auto wlan0 
iface wlan0 inet dhcp 
    wpa-ssid "xxx" 
    wpa-psk "xxx"
$ sudo ifup wlan0
$ ping -c 5 www.yahoo.co.jp
```

### ファイアウォールの設定

`BeagleBone Black`には、外からUSB経由のSSH接続だけを許可します。

``` bash
$ sudo apt-get install ufw
$ sudo ufw enable
$ sudo ufw default DENY
Default incoming policy changed to 'deny'
(be sure to update your rules accordingly)
$ sudo ufw allow in on usb0
Rule added
Rule added (v6)
$ sudo ufw status
Status: active

To                         Action      From
--                         ------      ----
Anywhere on usb0           ALLOW       Anywhere
Anywhere (v6) on usb0      ALLOW       Anywhere (v6)
```

### その他Ubuntuの初期設定

Ubuntuのデフォルトのnanoは使いにくいので、vimをエディタに使います。

``` bash
$ sudo apt-get install vim
$ sudo update-alternatives --config editor
```

adminグループは、sudoでパスワードを不要にします。

``` bash
$ sudo -i
# visudo
#%admin ALL=(ALL) ALL
%sudo   ALL=(ALL:ALL) ALL
%admin ALL=(ALL) NOPASSWD:ALL
```

公開鍵を設定します。

``` bash
$ vi ./ssh/authorized_keys
$ chmod 600 ./ssh/authorized_keys
```

タイムゾーンをAsia/Tokyoに設定します。
``` bash
$ sudo dpkg-reconfigure tzdata 
```

### まとめ

2時間くらいでUbuntuのインストール作業が終了しました。`BeagleBone Black`は小さいのでケースに入れて持ち運びも簡単です。USBケーブルが1本あれば電源供給とネットワーク接続ができるのもポイントです。Chromebookも含めてLinux-ARMは消費電力が少なく安価なハードウェアで動作するため、アイデア次第でいろんなことができそうな感じです。
