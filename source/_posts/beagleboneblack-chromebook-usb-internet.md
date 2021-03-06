title: "BeagleBone BlackとHP Chromebook 11 - Part2: インターネットに接続する"
date: 2015-02-15 19:41:02
tags:
 - BeagleBoneBlack
 - Chromebook
 - Ubuntu
description: 前回BeagleBone BlackとChromebookをUSB接続できるようになりました。しかしこの状態ではBeableBone Blackからインターネットに接続できません。Chromebookをルーターとして機能させるためにIPフォワードとファイアウォールの設定をします。
---

[前回](/2015/02/06/beagleboneblack-chromebook-usb-connect/)BeagleBone BlackとChromebookをUSB接続できるようになりました。しかしこの状態ではBeableBone Blackからインターネットに接続できません。Chromebookをルーターとして機能させるためにIPフォワードとファイアウォールの設定をします。

<!-- more -->

## Chromebookの設定

[BeagleBone Black USB Ethernet definitions](http://www.rpural.net/BlackNetworking)を参考にChromebookの設定をします。


`ctrl + alt + t`でcrosh > shellを起動します。

``` bash
crosh> shell
```

BeagleBone Blackを接続した状態でifconfigを確認します。`eth0`がBeagleBone BlackからUSB-Ethernet接続するインタフェースで、`mlan0`がChromebookからインターネットへつながる無線LANのインタフェースです。

``` bash
$ ifconfig
eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.7.1  netmask 255.255.255.252  broadcast 192.168.7.3
        inet6 fe80::9259:afff:fe58:32d3  prefixlen 64  scopeid 0x20<link>
        ether 90:59:af:58:32:d3  txqueuelen 1000  (Ethernet)
        RX packets 208  bytes 24036 (23.4 KiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 89  bytes 11263 (10.9 KiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        inet6 ::1  prefixlen 128  scopeid 0x10<host>
        loop  txqueuelen 0  (Local Loopback)
        RX packets 202  bytes 13036 (12.7 KiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 202  bytes 13036 (12.7 KiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

mlan0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.1.12  netmask 255.255.255.0  broadcast 192.168.1.255
        inet6 fe80::d2e7:82ff:feba:ddb7  prefixlen 64  scopeid 0x20<link>
        ether d0:e7:82:ba:dd:b7  txqueuelen 1000  (Ethernet)
        RX packets 2485  bytes 1233319 (1.1 MiB)
        RX errors 0  dropped 24  overruns 0  frame 0
        TX packets 2404  bytes 456542 (445.8 KiB)
        TX errors 4  dropped 0 overruns 0  carrier 0  collisions 0
```

ChromebookのIPフォワードを有効にします。

``` bash
$ echo 1 | sudo tee /proc/sys/net/ipv4/ip_forward > /dev/null
```

iptablesのルールを追加してeth0からmlan0を経由してインターネットへ接続できるようにNATします。

``` bash
$ sudo iptables -t nat -A POSTROUTING -o mlan0 -j MASQUERADE
```

mlan0とeth0の間は双方向でフォワードを許可します。

``` bash
$ sudo iptables -A FORWARD -i mlan0 -o eth0 -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
$ sudo iptables -A FORWARD -i eth0 -o mlan0 -j ACCEPT
```

## BeagleBone BlackのUbuntu 14.04 (SDカード)

Chromebookから BeagleBone BlackにSSH接続します。

* username: ubuntu
* password: temppwd

``` bash
$ ssh ubuntu@192.168.7.2
```

usb0インタフェースの設定は以下のように静的に定義しています。

``` bash /etc/network/interfaces
...
auto usb0
iface usb0 inet static
    address 192.168.7.2
    netmask 255.255.255.252
    network 192.168.7.0
    gateway 192.168.7.1
```

ルーティングを確認します。

``` bash
$ route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         192.168.7.1     0.0.0.0         UG    0      0        0 usb0
192.168.7.0     0.0.0.0         255.255.255.252 U     0      0        0 usb0
```

デフォルトゲートウェイの設定は正常に読み込まれていますが、表示されない場合は以下のように追加します。

``` bash
$ sudo route add default gw 192.168.7.1
```

`/etc/resolvconf/resolv.conf.d/base`にGoogleのDNSを追加します。

```bash /etc/resolvconf/resolv.conf.d/base
nameserver 8.8.8.8
nameserver 8.8.4.4
```

`/etc/resolv.conf`に反映させます。

``` bash
$ sudo resolvconf -u
$ cat /etc/resolv.conf
# Dynamic resolv.conf(5) file for glibc resolver(3) generated by resolvconf(8)
#     DO NOT EDIT THIS FILE BY HAND -- YOUR CHANGES WILL BE OVERWRITTEN
nameserver 8.8.8.8
nameserver 8.8.4.4
```


pingで名前解決とインターネット接続を確認します。

``` bash
$ ping -c 1 www.yahoo.co.jp
PING www.g.yahoo.co.jp (182.22.70.252) 56(84) bytes of data.
64 bytes from f5.top.vip.ssk.yahoo.co.jp (182.22.70.252): icmp_seq=1 ttl=54 time=9.71 ms

--- www.g.yahoo.co.jp ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 9.718/9.718/9.718/0.000 ms
```

BeagleBone Blackはバッテリがないので電源を入れたあとはシステムクロックの時間あわせをします。

``` bash
$ ntpdate -b -s -u pool.ntp.org
```
