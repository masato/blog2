title: "BeagleBone Blackのファームウェアを入れ直す - Part4: SDカードのUbuntu14.04.1とOSXを接続する"
date: 2015-01-31 21:21:02
tags:
 - Ubuntu
 - BeagleBoneBlack
 - Linux-ARM
 - OSX
description: いままでWindwos7をBeagleBone BlackのUSB-Ethernetネットワーク共有環境にして作業していました。OSXからも同様にUSBケーブルを接続して、SSHログインとインターネット接続を確認してみます。OSXのバージョンはYosemiteの10.10.1です。
---

これまでWindwos7をBeagleBone BlackのUSB-Ethernetネットワーク共有環境にして作業していました。OSXからも同様にUSBケーブルを接続して、SSHログインとインターネット接続を確認してみます。OSXのバージョンはYosemiteの10.10.1です。


<!-- more -->

## OSXにドライバをインストール

OSXの場合もWindowsと同様に[Getting Started](http://beagleboard.org/getting-started)を読みながらBeagleBone Blackと接続するためのドライバをインストールします。OSXの場合はNetworkドライバとSerialドライバの2つをインストールします。

### Networkドライバ

[Getting Started](http://beagleboard.org/getting-started)のOSXのリンクから、[HoRNDIS.pkg](http://beagleboard.org/static/Drivers/MacOSX/RNDIS/HoRNDIS.pkg)をダウンロードしてインストールします。インターネット接続使うUSBテザリングのドライバです。

### USB-シリアル変換ドライバ

[EnergiaFTDIDrivers2.2.18.pkg](http://beagleboard.org/static/Drivers/MacOSX/FTDI/EnergiaFTDIDrivers2.2.18.pkg)をダウンロードします。pkgはダブルクリックでなく、右クリックで開きます。

## ネットワーク設定

### USBケーブルで接続

ドライバをインストールしたら、BeagleBone BlackとOSXをUSBケーブルで接続します。コントロールパネルのネットワークを開いて接続を確認します。

![osx_network.png](/2015/01/31/beagleboneblack-ubuntu14-04-osx-yosemite/osx-network.png)

### 共有設定

コントロールパネルの共有設定を開き、BeagleBone Blackのネットワークと共有設定をします。

* コントロールパネル > 共有
* 共有する接続経路: Wi-Fi
* 相手のコンピュータが使用するポート: BeagleBoneBlack

左ペインの`インターネット共有`にチェックを入れます。

![network-share.png](/2015/01/31/beagleboneblack-ubuntu14-04-osx-yosemite/network-share.png)

## SSH接続

### OSXのネットワーク確認

OSXで共有設定をすると`bridge100`のネットワークが作成されました。

``` bash
$ ifconfig
...
bridge100: flags=8863<UP,BROADCAST,SMART,RUNNING,SIMPLEX,MULTICAST> mtu 1500
        options=3<RXCSUM,TXCSUM>
        ether 3e:15:c2:3d:d3:64
        inet 192.168.2.1 netmask 0xffffff00 broadcast 192.168.2.255
        inet6 fe80::3c15:c2ff:fe3d:d364%bridge100 prefixlen 64 scopeid 0xb
        Configuration:
                id 0:0:0:0:0:0 priority 0 hellotime 0 fwddelay 0
                maxage 0 holdcnt 0 proto stp maxaddr 100 timeout 1200
                root id 0:0:0:0:0:0 priority 0 ifcost 0 port 0
                ipfilter disabled flags 0x2
        member: en5 flags=3<LEARNING,DISCOVER>
                ifmaxaddr 0 port 10 priority 0 path cost 0
        nd6 options=1<PERFORMNUD>
        media: autoselect
        status: active
```

OSXでネットワークの共有設定をする場合、サブネットは`192.168.2/24`になっています。

```xml /etc/bootpd.plist
...
			<key>dhcp_router</key>
			<string>192.168.2.1</string>
			<key>interface</key>
			<string>bridge100</string>
...
			<key>name</key>
			<string>192.168.2/24</string>
			<key>net_address</key>
			<string>192.168.2.0</string>
			<key>net_mask</key>
			<string>255.255.255.0</string>
			<key>net_range</key>
```

### BeagleBone Blackのネットワーク確認

WindowsのPuttyからUSB-シリアル接続をして、dhclientを実行します。

``` bash
$ sudo dhclient -v
Internet Systems Consortium DHCP Client 4.2.4
Copyright 2004-2012 Internet Systems Consortium.
All rights reserved.
For info, please visit https://www.isc.org/software/dhcp/

Listening on LPF/usb0/02:e9:fb:f0:0b:f0
Sending on   LPF/usb0/02:e9:fb:f0:0b:f0
Sending on   Socket/fallback
DHCPREQUEST of 192.168.2.9 on usb0 to 255.255.255.255 port 67 (xid=0x137b281b)
DHCPACK of 192.168.2.9 from 192.168.2.1
RTNETLINK answers: File exists
bound to 192.168.2.9 -- renewal in 42520 seconds.
```

ルーティングを確認すると、サブネットが変更されました。

``` bash
$ route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         192.168.2.1     0.0.0.0         UG    0      0        0 usb0
192.168.2.0     0.0.0.0         255.255.255.0   U     0      0        0 usb0
```

OSXのターミナルからBeagleBone BlackにSSH接続します。

``` bash
$ ssh debian@192.168.2.2
Ubuntu 14.04.1 LTS

rcn-ee.net console Ubuntu Image 2015-01-06

Support/FAQ: http://elinux.org/BeagleBoardUbuntu

default username:password is [ubuntu:temppwd]

ubuntu@192.168.2.2's password:
Welcome to Ubuntu 14.04.1 LTS (GNU/Linux 3.14.31-ti-r49 armv7l)
```

`usb0`ネットワークを確認します。OSXの`bridge100`と同じネットワークになっています。

``` bash
$ ip addr show usb0
5: usb0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 2e:3d:e9:00:de:ca brd ff:ff:ff:ff:ff:ff
    inet 192.168.2.2/24 brd 192.168.2.255 scope global usb0
       valid_lft forever preferred_lft forever
    inet6 fe80::2c3d:e9ff:fe00:deca/64 scope link
       valid_lft forever preferred_lft forever
``` 

BeagleBone BlackからOSXのUSB-Ethernet接続を経由して、名前解決とインターネットに接続ができるようになりました。

```
$ ping -c 3 www.yahoo.co.jp
PING www.g.yahoo.co.jp (183.79.235.148) 56(84) bytes of data.
64 bytes from 183.79.235.148: icmp_seq=1 ttl=52 time=18.1 ms
...
```
