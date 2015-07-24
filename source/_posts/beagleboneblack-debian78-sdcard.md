title: 'BeagleBone Blackで1-WireセンサのDS18B20を使う - Part2: SDカードにDebian7.8をインストールする'
date: 2015-07-10 22:36:29
categories:
 - IoT
tags:
 - DS18B20
 - BeagleBoneBlack
 - Debian
description: BeagleBone BlackのeMMCにはDebianが入っていますが、今までSDカードはUbuntuを入れて使っていました。3.8.13のカーネルが必要なのでDebian 7.8を使おうと思います。久しぶりにSDカードを作成したら結構嵌まりました。
---

BeagleBone BlackのeMMCにはDebianが入っていますが、今までSDカードはUbuntuを入れて使っていました。3.8.13のカーネルが必要なのでDebian 7.8を使おうと思います。久しぶりにSDカードを作成したら結構嵌まりました。


<!-- more -->

## SDカードへインストール

作業マシンにMacBook Proを使います。Windowsだとイメージの書き込みにすごく時間がかかった記憶がありますが、MacBook Proだと数分でSDカードに書き込みが終了しました。

### ダウンロード

イメージは[2015-07-06](http://elinux.org/Beagleboard:BeagleBoneBlack_Debian#2015-07-06)にリリースされたばかりの最新版をダウンロードします。

```bash
$ cd ~/Dowonloads
$ wget https://rcn-ee.com/rootfs/bb.org/release/2015-07-06/console/bone-debian-7.8-console-armhf-2015-07-06-2gb.img.xz
```

チェックサムを確認します。

```bash
$ openssl md5 bone-debian-7.8-console-armhf-2015-07-06-2gb.img.xz
MD5(bone-debian-7.8-console-armhf-2015-07-06-2gb.img.xz)= 6aeef824652637602dbdb1eae431203a
```

### unarでイメージを解凍する

[Unarchiver](http://unarchiver.c3.cx/commandline)のコマンドラインをインストールします。

```bash
$ wget http://theunarchiver.googlecode.com/files/unar1.8.1.zip
$ unzip unar1.8.1.zip -d ~/bin
Archive:  unar1.8.1.zip
  inflating: /Users/mshimizu/bin/lsar
  inflating: /Users/mshimizu/bin/unar
```

unarコマンドを使ってxz形式の圧縮ファイルを解凍します。

```bash
$ unar bone-debian-7.8-console-armhf-2015-07-06-2gb.img.xz
bone-debian-7.8-console-armhf-2015-07-06-2gb.img.xz: XZ
  bone-debian-7.8-console-armhf-2015-07-06-2gb.img... OK.
Successfully extracted to "./bone-debian-7.8-console-armhf-2015-07-06-2gb.img".
```

## ddコマンドで書き込む

SDカードをMacBook Proに挿してdfコマンドからデバイス名を確認します。

```bash
$ df -h
Filesystem      Size   Used  Avail Capacity  iused    ifree %iused  Mounted on
/dev/disk1     465Gi   86Gi  379Gi    19% 22550671 99288943   19%   /
devfs          186Ki  186Ki    0Bi   100%      645        0  100%   /dev
map -hosts       0Bi    0Bi    0Bi   100%        0        0  100%   /net
map auto_home    0Bi    0Bi    0Bi   100%        0        0  100%   /home
/dev/disk2s1    15Gi  2.3Mi   15Gi     1%        0        0  100%   /Volumes/NO NAME
```

作業前にSDカードをdiskutilでunmountします。

```bash
$ sudo diskutil unmount /dev/disk2s1
Volume NO NAME on disk2s1 unmounted
```

ddコマンドでSDカードのraw diskを指定して書き込みます。`/dev/disk2s1`の場合、raw disk は`/dev/rdisk2`になります。

```bash
$ sudo dd bs=1m if=bone-debian-7.8-console-armhf-2015-07-06-2gb.img of=/dev/rdisk2
1700+0 records in
1700+0 records out
1782579200 bytes transferred in 147.239980 secs (12106625 bytes/sec)
```

書き込みには数分かかります。終了するとOSXが読み取れないディスクとダイアログを出しますが無視できます。diskutilコマンドでSDカードを取り出します。

```bash
$ sudo diskutil eject /dev/rdisk2
Disk /dev/rdisk2 ejected
```

### シリアル接続で確認

OSXからシリアル接続して起動を確認します。デフォルトのusername:passwordは`debian:temppwd`です。

```bash
BeagleBoard.org Debian Image 2015-07-06

Support/FAQ: http://elinux.org/Beagleboard:BeagleBoneBlack_Debian

default username:password is [debian:temppwd]

beaglebone login:
Password:
Linux beaglebone 3.8.13-bone72 #1 SMP Tue Jun 16 21:36:04 UTC 2015 armv7l

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
debian@beaglebone:~$
```

## 有線LANが必要

このDebian 7.8のイメージはかなり厳密にできていて、デフォルトでは無線LANに接続するツールが入っていません。OSXを経由したUSBテザリングがうまく動作しなくなったので、BeagleBone Blackに有線LANをつないで必要なパッケージをインストールします。

``` bash
$ sudo apt-get update
$ sudo apt-get install usbutils wpasupplicant wireless-tools
```

ネットワークを再起動します。

```bash
$ sudo systemctl restart networking.service
```

lsusbコマンドで無線LANのドングルが認識されていることを確認します。

```bash
$ lsusb
Bus 001 Device 002: ID 2019:ab2a PLANEX GW-USNano2 802.11n Wireless Adapter [Realtek RTL8188CUS]
Bus 001 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub
Bus 002 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub
```


## 無線LANの設定

wpa_passphraseコマンドにSSIDとパスワードを指定して、wpa_supplicant.confファイルを作成します。

```bash
$ sudo sh -c 'wpa_passphrase SSID PASSWORD >> /etc/wpa_supplicant/wpa_supplicant.conf'
$ sudo vi /etc/wpa_supplicant/wpa_supplicant.conf
network={
  ssid="SSID"
  #psk="PASSWORD"
  psk=PASSWORD
}
```

/etc/network/interfacesファイルにwlan0の起動とwpa_supplicant.confの設定を読む設定をします。

```bash
$ sudo vi /etc/network/interfaces
...
allow-hotplug wlan0
auto wlan0
iface wlan0 inet dhcp
wpa-conf /etc/wpa_supplicant/wpa_supplicant.conf
iface default inet dhcp
...
```

wlan0を再起動してDHCPからIPアドレスを取得します。

```bash
$ sudo ifdown wlan0
$ sudo ifup wlan0
```

最後にpingで名前解決とインターネットへの接続を確認します。

```bash
$ ping -c 1 www.yahoo.co.jp
PING www.g.yahoo.co.jp (183.79.71.173) 56(84) bytes of data.
64 bytes from f13.top.vip.kks.yahoo.co.jp (183.79.71.173): icmp_req=1 ttl=53 time=23.2 ms

--- www.g.yahoo.co.jp ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 23.215/23.215/23.215/0.000 ms
```

## タイムゾーンの変更

初期設定としてタイムゾーンを`Asia/Tokyo`にしておきます。

```bash
$ sudo cp /etc/localtime{,.orig}
$ sudo ln -sf  /usr/share/zoneinfo/Asia/Tokyo /etc/localtime
```

## 日付

ntpをインストールします。

```bash
$ sudo apt-get install ntp
```

NICTのNTPサーバーを設定します。

```bash /etc/ntp.conf
#server 0.debian.pool.ntp.org iburst
#server 1.debian.pool.ntp.org iburst
#server 2.debian.pool.ntp.org iburst
#server 3.debian.pool.ntp.org iburst
pool ntp.nict.jp iburst
```

ntpdの再起動をします。

```bash
$ systemctl start ntp.service
```

## Vim

エディタのVimをインストールします。

```bash
$ sudo apt-get install vim
$ mkdir ~/tmp
```

バックアップを作らない設定だけします。

```bash ~/.vimrc
set backupdir=~/tmp
```