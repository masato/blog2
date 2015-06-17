title: "Intel EdisonからChromebookのUSB tetheringを使う"
date: 2015-03-05 13:57:45
tags:
 - Chromebook
 - IntelEdison
 - USBtethering
description: スイッチサイエンスでIntel Edison Kit for Arduinoが売り切れだったので、Intel Edison Breakout Board Kitを購入しました。最初はLinuxサーバーとして使うのでArduinoは要らないと思っていたのですが、Grove Starter Kit for Arduinoを見ていたらArduinoとつなぎたくなります。Intel Edison Board for Arduinoが単体でも売っているようなのでどこかで入手しようと思います。
---

スイッチサイエンスで[Intel Edison Kit for Arduino](https://www.switch-science.com/catalog/1958/)が売り切れだったので、[Intel Edison Breakout Board Kit](https://www.switch-science.com/catalog/1957/)を購入しました。最初はLinuxサーバーとして使うのでArduinoは要らないと思っていたのですが、[Grove Starter Kit for Arduino](http://www.seeedstudio.com/depot/Grove-Starter-Kit-for-Arduino-p-1855.html)を見ていたらArduinoとつなぎたくなります。[Intel Edison Board for Arduino](http://ark.intel.com/ja/products/84574/Intel-Edison-Board-for-Arduino)が単体でも売っているようなのでどこかで入手しようと思います。

<!-- more -->

## Chromebookの設定

Chromebookは先日[電源が入らない問題](/2015/03/04/hp-chromebook-11-wont-turn-on/)から復旧したばかりの、HP Chromebook 11を使います。Chromeブラウザから`Ctrl + Alt + t`をタイプしてcroshを起動します。

``` bash
crosh> shell
chronos@localhost
```

### Intel EdisonはJ16のUSBコネクタと接続する

USB-Serial接続する場合はJ3のUSBコネクタに接続します。Intel EdisonのmicroUSBコネクタの近くにプリントされているので間違えないようにします。デバイスファイルは`/dev/ttyUSB0`が作成されます。croutonでインストールしたUbuntuからは以下のようにIntel Edisonとシリアル接続ができます。

``` bash
$ sudo screen /dev/ttyUSB0 115200
```
USB tetheringを使う場合はJ16のポートを使います。デバイスファイルは`/dev/ttyACM0`になります。USB tetheringの方法はBeagleBone BlackをChromebookに接続したときと同じです。まずifconfigでChromebookのネットワークデバイスを確認します。

``` bash
$ ifconfig
lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        inet6 ::1  prefixlen 128  scopeid 0x10<host>
        loop  txqueuelen 0  (Local Loopback)
        RX packets 79419  bytes 5338895 (5.0 MiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 79419  bytes 5338895 (5.0 MiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

mlan0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 10.11.41.155  netmask 255.255.255.0  broadcast 10.11.41.255
        inet6 fe80::d2e7:82ff:feba:ddb7  prefixlen 64  scopeid 0x20<link>
        ether d0:e7:82:ba:dd:b7  txqueuelen 1000  (Ethernet)
        RX packets 28479  bytes 23468244 (22.3 MiB)
        RX errors 0  dropped 743  overruns 0  frame 0
        TX packets 19001  bytes 3423153 (3.2 MiB)
        TX errors 6  dropped 0 overruns 0  carrier 0  collisions 0

usb0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet6 fe80::b0fd:5cff:fe2c:350c  prefixlen 64  scopeid 0x20<link>
        ether b2:fd:5c:2c:35:0c  txqueuelen 1000  (Ethernet)
        RX packets 31  bytes 5275 (5.1 KiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 44  bytes 9880 (9.6 KiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```

Intel EdisonとUSBケーブルで接続したusb0が認識されていますがIPアドレスは振られていません。mlan0を通してChromebookはインターネットに接続します。

### IPフォワードを有効にする

ChromebookのIPフォワードを有効にします。

``` bash
$ sudo sh -c 'echo 1 >> /proc/sys/net/ipv4/ip_forward'
```

usb0にIPアドレスを設定します。

``` bash
$ sudo ifconfig usb0 192.168.2.1 netmask 255.255.255.0
$ ip addr show usb0
4: usb0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UNKNOWN qlen 1000
    link/ether b2:fd:5c:2c:35:0c brd ff:ff:ff:ff:ff:ff
    inet 192.168.2.1/24 brd 192.168.2.255 scope global usb0
```

Intel EdisonからChromebookを通してインターネットに接続できるように、mlan0とusb0の間にFORWARDのルールを設定します。

``` bash
$ sudo iptables -t nat -A POSTROUTING -o mlan0 -j MASQUERADE
$ sudo iptables -A FORWARD -i mlan0 -o usb0 -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
$ sudo iptables -A FORWARD -i usb0 -o mlan0 -j ACCEPT
```

Intel EdisonにChromebookからSSH接続します。デフォルトでrootユーザーにパスワードは設定されていません。

* username: root
* password: なし

``` bash
$ ssh root@192.168.2.15
root@edison:~#
```

## Intel Edisonの設定

とりあえず現在のバージョンを確認してみます。

``` bash
$ cat /etc/version 
edison-weekly_build_56_2014-08-20_15-54-05
```

Intel Edisonで名前解決ができるようにresolv.confにGoogle DNSを追加します。

``` bash
$ echo "nameserver 8.8.8.8" >> /etc/resolv.conf
```

デフォルトゲートウェイを設定してrouteとusb0の状態を確認します。

``` bash
$ route add default gw 192.168.2.1
$ route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         192.168.2.1     0.0.0.0         UG    0      0        0 usb0
192.168.2.0     0.0.0.0         255.255.255.0   U     0      0        0 usb0
$ ip addr show usb0
4: usb0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast qlen 1000
    link/ether 1e:8b:02:ec:09:1d brd ff:ff:ff:ff:ff:ff
    inet 192.168.2.15/24 brd 192.168.2.255 scope global usb0
       valid_lft forever preferred_lft forever
    inet6 fe80::1c8b:2ff:feec:91d/64 scope link 
       valid_lft forever preferred_lft forever
```

pingで名前解決とインターネット接続の確認ができました。

``` bash
$ ping -c 1 www.yahoo.co.jp
PING www.yahoo.co.jp (124.83.183.243): 56 data bytes
64 bytes from 124.83.183.243: seq=0 ttl=49 time=14.356 ms

--- www.yahoo.co.jp ping statistics ---
1 packets transmitted, 1 packets received, 0% packet loss
round-trip min/avg/max = 14.356/14.356/14.356 ms
```
