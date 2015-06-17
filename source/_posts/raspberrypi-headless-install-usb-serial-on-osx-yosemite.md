title: "Raspberry PiをOSX YosemiteとUSB-Serial接続してヘッドレスインストールと無線LANの設定"
date: 2015-02-26 23:49:54
categories:
 - IoT
tags:
 - RaspberryPi
 - OSX
 - Yosemite
description: 2年くらい前に購入したRaspberry Pi Model BのRaspbianを新しくしようと思います。OSX YosemiteにはBeagleBone Blackとの接続用にUSB-Serialドライバがインストールしてあります。Raspberry Piも同様にディスプレイとキーボードを接続しないで、OSXとUSB-Serial接続をしてヘッドレスインストールします。
---

2年くらい前に購入したRaspberry Pi Model BのRaspbianを新しくしようと思います。OSX YosemiteにはBeagleBone Blackとの接続用にUSB-Serialドライバがインストールしてあります。Raspberry Piも同様にディスプレイとキーボードを接続しないで、OSXとUSB-Serial接続をしてヘッドレスインストールします。

<!-- more -->


## SDカードにRaspbianイメージを焼く

[Downloads](http://www.raspberrypi.org/downloads/)ページからRaspbianイメージをダウンロードします。バージョンは`February 2015`です。

``` bash
$ cd ~/Downloads
$ wget -O 2015-02-16-raspbian-wheezy.zip http://downloads.raspberrypi.org/raspbian_latest
$ unzip 2015-02-16-raspbian-wheezy.zip
Archive:  2015-02-16-raspbian-wheezy.zip
  inflating: 2015-02-16-raspbian-wheezy.img
```

diskutilコマンドでイメージを焼くためのSDカードのデバイスを確認します。SDカードを挿しているデバイスは`/dev/disk2`です。SDカードには以前インストールしてあったBeagleBone Blackのが入ったままでした。

``` bash
$ diskutil list
...
/dev/disk2
   #:                       TYPE NAME                    SIZE       IDENTIFIER
   0:     FDisk_partition_scheme                        *15.9 GB    disk2
   1:             Windows_FAT_16 BOOT                    100.7 MB   disk2s1
   2:                      Linux                         1.7 GB     disk2s2
```

diskutilコマンドでSDカードをアンマウントしてからddコマンドでイメージを焼きます。microSDカードをアダプタに挿して使っているためか、2時間くらいかかりました。

``` bash
$ diskutil unmountDisk /dev/disk2
Unmount of all volumes on disk2 was successful
$ sudo dd if=~/Downloads/2015-02-16-raspbian-wheezy.img of=/dev/disk2 bs=4m
```

## OSXとUSB-Serial接続

Raspberry PiをヘッドレスインストールするためにOSXとUSB-Serial接続できるようにします。シリアル接続については以下のサイトを参考にします。

* [Software Installation (Mac)](https://learn.adafruit.com/adafruits-raspberry-pi-lesson-5-using-a-console-cable?view=all)
* [FIX USB SERIAL CONSOLE ON RASPBERRY PI FOR YOSEMITE]( http://zittlau.ca/fix-usb-serial-console-on-raspberry-pi-for-yosemite/)

### ドライバのインストール
 
すでにOSXには[BeagleBone Black](http://beagleboard.org/getting-started)を接続しているので、[FTDIのVCPドライバ](http://www.ftdichip.com/Drivers/VCP.htm)の[EnergiaFTDIDrivers2.2.18](http://beagleboard.org/static/Drivers/MacOSX/FTDI/EnergiaFTDIDrivers2.2.18.pkg)をインストールしています。

### USB-Serial変換

[PL2303HX内蔵USB-Serial変換ケーブル](http://www.amazon.co.jp/gp/product/B00L8SP7U6)はアマゾンから購入しました。[Software Installation (Mac)]( https://learn.adafruit.com/adafruits-raspberry-pi-lesson-5-using-a-console-cable?view=all)を参考にしてGPIOピンにつなぎます。

* 赤: 5V
* 黒: GND
* 白: RXD
* 緑: TXD

![learn_raspberry_pi_gpio_closeup.jpg](/2015/02/26/raspberrypi-headless-install-usb-serial-on-osx-yosemite/learn_raspberry_pi_gpio_closeup.jpg)

### USB-Serial接続

OSXからシリアル接続する場合はターミナルから以下を実行します。Yosemiteの場合デバイスファイル名は`cu.usbserial`です。

``` bash
$ screen /dev/cu.usbserial 115200
```

ちなみにChromebookのUbuntuからシリアル接続する場合は以下です。

``` bash
$ sudo screen /dev/ttyUSB0 115200
```

デフォルトのusernameとpasswordでログインします。

* username: pi
* password: raspberry

``` bash
Raspbian GNU/Linux 7 raspberrypi ttyAMA0

raspberrypi login: pi
Password: 
Linux raspberrypi 3.18.7+ #755 PREEMPT Thu Feb 12 17:14:31 GMT 2015 armv6l
...
NOTICE: the software on this Raspberry Pi has not been fully configured. Please run 'sudo raspi-config'
```

## 初期設定

コンソールに接続したらガイドに従って初期設定をします。

``` bash
$ sudo raspi-config
```

オプションのリストが表示されるまでにちょっと時間がかかりました。

* Expand Filesystem > Select
* Internationalisation Options > Change Locale > ja_JP.UTF-8
* Internationalisation Options > Change Timezone > Asia > Tokyo


## 無線LAN

無線LANのUSBドングルは[BUFFALO WLI-UC-GNM](http://www.amazon.co.jp/dp/B003NSAMW2)を使います。


### 電源不足

USB-Serialケーブルは給電もしてくれますが、無線LANを使う時は電源不足になるので別途microUSB電源ケーブルを接続します。USB ACアダプタは1.8Aくらいないと`apt-get update`したときにRaspberry Piが落ちてしまします。

### アクセスポイント検知 - wpa_supplicant.confを使わない場合

最初はアクセスポイントの検知ができる無線LAN環境での設定です。wpa_supplicant.confを使わずに`/etc/network/interfaces`にSSIDのパスフレーズを直接書きます。

``` bash /etc/network/interfaces
auto lo

iface lo inet loopback
iface eth0 inet dhcp

auto wlan0
iface wlan0 inet dhcp
  wpa-ssid "ssid"
  wpa-psk "passphrase"
```

NICを再起動します。

``` bash
$ sudo ifdown wlan0
$ sudo ifup wlan0
```

IPアドレスがDHCPから取得できました。

``` bash
$ ip addr show wlan0
7: wlan0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP qlen 1000
    link/ether 4c:e6:76:42:4f:a3 brd ff:ff:ff:ff:ff:ff
    inet 192.168.1.8/24 brd 192.168.1.255 scope global wlan0
       valid_lft forever preferred_lft forever
```

名前解決とインターネット接続を確認します。

``` bash
$ ping -c 1 www.yahoo.co.jp
PING www.g.yahoo.co.jp (183.79.11.230) 56(84) bytes of data.
64 bytes from f6.top.vip.kks.yahoo.co.jp (183.79.11.230): icmp_req=1 ttl=52 time=17.8 ms

--- www.g.yahoo.co.jp ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 17.869/17.869/17.869/0.000 ms
```

### アクセスポイント検知 - wpa_supplicant.confを使わない場合

wpa_supplicant.confファイルを使う場合は、`/etc/network/interfaces`から読み込むようにします。今回はEthernetポートは使わずにUSBドングルの無線LANだけをつかうので、wpa-roamでなくwpa-confのディレクティブで指定する必要があります。

``` bash /etc/network/interfaces
auto lo

iface lo inet loopback
iface eth0 inet dhcp

auto wlan0
allow-hotplug wlan0
iface wlan0 inet dhcp
wpa-conf /etc/wpa_supplicant/wpa_supplicant.conf
iface default inet dhcp
```

wpa_passphraseコマンドを使いwpa_supplicant.confファイルを作成します。

``` bash
$ sudo sh -c "wpa_passphrase $1 $2 >> /etc/wpa_supplicant/wpa_supplicant.conf"
```

wpa_supplicant.confには平文でパスフレーズがコメントアウトされた状態で残っているので削除しておきます。

``` bash /etc/wpa_supplicant/wpa_supplicant.conf
ctrl_interface=DIR=/var/run/wpa_supplicant GROUP=netdev
update_config=1
network={
  ssid="ssid"
  proto=WPA2
  key_mgmt=WPA-PSK
  psk=passphrase
}
```

### ステルスモード - wpa_supplicant.confを使わない場合

無線LANのアクセスポイントがステルスモードの場合は`wpa-scan-ssid 1`を追加します。

``` bash /etc/network/interfaces
auto lo

iface lo inet loopback
iface eth0 inet dhcp

auto wlan0
allow-hotplug wlan0
iface wlan0 inet dhcp
wpa-scan-ssid 1
wpa-ssid "ssid"
wpa-psk "passphrase"
iface default inet dhcp
```

### ステルスモード - wpa_supplicant.confを使う場合

同様にwpa_supplicant.confは`scan_ssid=1`を追加します。

``` bash /etc/wpa_supplicant/wpa_supplicant.conf
ctrl_interface=DIR=/var/run/wpa_supplicant GROUP=netdev
update_config=1
network={
  ssid="ssid"
  scan_ssid=1
  proto=WPA2
  key_mgmt=WPA-PSK
  psk=passphrase
}
```

`/etc/network/interfaces`はwpa-confディレクティブでwpa_supplicant.confを読み込むようにします。

``` bash /etc/network/interfaces
auto lo

iface lo inet loopback
iface eth0 inet dhcp

auto wlan0
allow-hotplug wlan0
iface wlan0 inet dhcp
wpa-conf /etc/wpa_supplicant/wpa_supplicant.conf
iface default inet dhcp
```


