title: "BeagleBone BlackとHP Chromebook 11 - Part1: USBで接続する"
date: 2015-02-06 23:02:19
tags:
 - BeagleBoneBlack
 - Chromebook
description: BeagleBone Blackの開発用端末としてChromebookを使うためにUSBケーブルで接続できるようにします。USB-Ethernet接続はケーブルをつなぐだけです。USB-Serial接続をする場合は、ドライバのインストールとscreenコマンドが必要なのでChromeOSからは直接使えません。
---

* Update: [BeagleBone BlackとHP Chromebook 11 - Part2: インターネットに接続する](/2015/02/15/beagleboneblack-chromebook-usb-internet/)

BeagleBone Blackの開発用端末としてChromebookを使うためにUSBケーブルで接続できるようにします。USB-Ethernet接続はケーブルをつなぐだけです。USB-Serial接続をする場合は、ドライバのインストールとscreenコマンドが必要なのでChromeOSからは直接使えません。

<!-- more -->

## USB-Ethernet接続

Chromeブラウザから`ctrl + alt + t`をタイプしてshellを実行します。

``` bash
crosh> shell
chronos@localhost / $ 
```

BeagleBone BlackのeMMCにインストールしたDebianは、ChromebookとUSBで接続すればそのまま使えます。

``` bash
crosh> shell
chronos@localhost / $ ssh debian@192.168.7.2
Debian GNU/Linux 7
```

SDカードからブートしたUbuntuも[カーネルを更新](/2015/02/01/beagleboneblack-ubuntu14-04-windows7/)しているので、USBケーブルを接続するだけSSH接続ができした。

``` bash
crosh> shell
chronos@localhost / $ ssh debian@192.168.7.2
Ubuntu 14.04.1 LTS

rcn-ee.net console Ubuntu Image 2015-01-06
```

## USB-Serial接続

USB-Serial接続する場合、[Getting Started](http://beagleboard.org/getting-started)のインストール手順に従ってLinux用ドライバをインストールします。


### croshからはインストールできない

Chromeブラウザから`ctrl + alt + t`をタイプしてshellを実行します。

``` bash
crosh> shell
chronos@localhost / $ 
```

ChromeOSのファイルシステムがRead-onlyなので、直接ChromeOSにはドライバをインストールできません。

``` bash
$ cd ~/Downloads
$ wget http://beagleboard.org/static/Drivers/Linux/FTDI/mkudevrule.sh
$ sudo sh -e mkudevrule.sh 
mkudevrule.sh: 2: cannot create /etc/udev/rules.d/73-beaglebone.rules: Read-only file system
```

### chrootしてインストールする 

[crouton](https://github.com/dnschneid/crouton)で[chrootにUbuntuをインストール](/2015/02/05/chromebook-ubuntu-trusty-extension/)しておきます。croshからenter-chrootします。

``` bash
$ sudo enter-chroot
$ cd ~/Downloads
$ sudo sh -e mkudevrule.sh 
```

USBドライバがインストールできたようです。

``` bash
$ cat /etc/udev/rules.d/73-beaglebone.rules
ACTION=="add", SUBSYSTEM=="usb", ENV{DEVTYPE}=="usb_interface",         ATTRS{idVendor}=="0403", ATTRS{idProduct}=="a6d0",         DRIVER=="", RUN+="/sbin/modprobe -b ftdi_sio"
...
```

USB-Serial接続する場合はsceenを使います。

``` bash
$ sudo apt-get update
$ sudo apt-get install screen
```

SDカードからブートしたUbuntuにscreenでシリアル通信ができようになりました。

``` bash
$ sudo screen /dev/ttyUSB0 115200
Ubuntu 14.04.1 LTS arm ttyO0

rcn-ee.net console Ubuntu Image 2015-01-06
```
