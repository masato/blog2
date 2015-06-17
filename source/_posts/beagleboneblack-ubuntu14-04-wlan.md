title: "BeagleBone Blackのファームウェアを入れ直す - Part3: SDカードのUbuntu14.04.1の無線LAN設定"
date: 2015-01-30 22:00:15
tags:
 - Ubuntu
 - BeagleBoneBlack
 - Linux-ARM
description: Ubuntu 14.04.1をSDカードから起動して、USB-シリアル接続ができるようになりましたが、USB Gadgetを使ったUSB-Ethernet接続ができません。無線LANの設定をしてインターネットに接続できるようにします。
---

[Ubuntu 14.04.1をSDカードから起動](/2015/01/29/beagleboneblack-ubuntu14-04/)して、USB-シリアル接続ができるようになりましたが、[USB Gadget](http://www.linux-usb.org/gadget/)を使ったUSB-Ethernet接続ができません。無線LANの設定をしてインターネットに接続できるようにします。

<!-- more -->

## 無線LANの設定

PuttyからUSB-シリアル接続をしてコンソールを表示します。BeagleBone Blackは[Anker 40W 5ポート](http://www.amazon.co.jp/dp/B00GTGETFG)などUSB電源アダプタに接続します。この状態だとインターネットに接続できないので、まずは無線LANの設定をします。`/etc/network/interfaces`にSSIDとキーを記述します。

```bash:/etc/network/interfaces
# WiFi Example
auto wlan0
iface wlan0 inet dhcp
    wpa-ssid "xxx"
    wpa-psk  "xxx"
```

無線LANのUSBドングルは余っていた[BUFFALO WLI-UC-GNM2](http://www.amazon.co.jp/dp/B005DU4XSM)を使いました。USBドングルを挿して無線LANの確認をします。

``` bash
$ sudo ifup wlan0
$ ip addr show wlan0
5: wlan0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether 4c:e6:76:42:4f:a3 brd ff:ff:ff:ff:ff:ff
    inet 192.168.1.8/24 brd 192.168.1.255 scope global wlan0
       valid_lft forever preferred_lft forever
    inet6 fe80::4ee6:76ff:fe42:4fa3/64 scope link
       valid_lft forever preferred_lft forever
```

名前解決もできています。

``` bash
$ ping -c 1 www.yahoo.co.jp
PING www.g.yahoo.co.jp (183.79.139.228) 56(84) bytes of data.
64 bytes from f2.top.vip.kks.yahoo.co.jp (183.79.139.228): icmp_seq=1 ttl=53 time=24.6 ms
...
```


## PuttyからSSH接続

Puttyから`192.168.1.8`にSSH接続することができました。

``` bash
Using username "ubuntu".
Ubuntu 14.04.1 LTS

rcn-ee.net console Ubuntu Image 2015-01-06

Support/FAQ: http://elinux.org/BeagleBoardUbuntu

default username:password is [ubuntu:temppwd]

ubuntu@192.168.1.8's password:
Welcome to Ubuntu 14.04.1 LTS (GNU/Linux 3.14.26-ti-r43 armv7l)
```
