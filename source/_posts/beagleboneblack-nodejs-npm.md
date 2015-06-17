title: 'BeagleBone BlackのUbuntu14.04.1にNode.jsとnpmをインストールする'
date: 2015-02-09 22:07:01
tags:
 - Ubuntu
 - BeagleBoneBlack
 - Nodejs
 - npm
description: 以前BeagleBone BlackのUbuntuにCloud9をインストールしたときは、Node.jsはARMhf.comからビルド済みにバイナリをダウンロードして使いました。現在ではDownloads (old)のサイトに移動してUbuntu Saucy用にNode.jsのv0.10.21がダウンロードできます。新しいバージョンはビルドされていないようです。
---

以前BeagleBone Blackの[UbuntuにCloud9をインストール](/2014/05/04/beagleboneblack-cloud9/)したときは、Node.jsは[ARMhf.com](http://www.armhf.com/)からビルド済みにバイナリをダウンロードして使いました。現在では[Downloads (old)](http://www.armhf.com/downloads-old/)のサイトに移動してUbuntu Saucy用にNode.jsのv0.10.21がダウンロードできます。新しいバージョンはビルドされていないようです。

<!-- more -->


## Windows7とUSB-Ethernet接続

BeagleBone BlackとWindows7のPCをUSBケーブルをつないでSSH接続します。デフォルトのusernameとpasswordは以下です。

* username: ubuntu
* password: temppwd

``` bash
$ ssh ubuntu@192.168.7.2
```

デフォルトゲートウェイを設定します。

``` bash
$ sudo route add default gw 192.168.7.1
$ route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         192.168.7.1     0.0.0.0         UG    0      0        0 usb0
192.168.7.0     0.0.0.0         255.255.255.252 U     0      0        0 usb0
$ ping -c 1 www.yahoo.co.jp
PING www.g.yahoo.co.jp (183.79.43.200) 56(84) bytes of data.
64 bytes from f5.top.vip.kks.yahoo.co.jp (183.79.43.200): icmp_seq=1 ttl=53 time=25.7 ms
```

## Node.jsのインストール

Ubuntuのバージョンを確認します。

``` bash
$ cat /etc/lsb-release
DISTRIB_ID=Ubuntu
DISTRIB_RELEASE=14.04
DISTRIB_CODENAME=trusty
DISTRIB_DESCRIPTION="Ubuntu 14.04.1 LTS"
```

apt-cacheでNode.jsとnpmのバージョンを確認します。少しバージョンは古くなりますが、野良ビルドすると時間がかかるのでパッケージマネージャを使います。

``` bash
$ sudo apt-get update
$ apt-cache show nodejs | grep Version
Version: 0.10.25~dfsg2-2ubuntu1
$ apt-cache show npm | grep Version
Version: 1.3.10~dfsg-1
```

apt-getからインストールします。`/usr/bin/node`からも実行できるように`update-alternatives`します。

``` bash
$ sudo apt-get install nodejs npm
$ sudo update-alternatives --install /usr/bin/node node /usr/bin/nodejs 10
```

Node.jsとnpmのインストールを確認します。

``` bash
$ node -v
v0.10.25
$ npm -v
1.3.10
```
