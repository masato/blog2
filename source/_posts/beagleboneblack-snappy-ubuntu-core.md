title: "Snappy Ubuntu Core on BeagleBone Black- Part1: インストール"
date: 2015-02-03 00:29:56
tags:
 - SnappyUbuntuCore
 - BeagleBoneBlack
 - Docker
 - IoT
 - NinjaSphere
description: CanonicalがSnappy Ubuntu CoreのIoTバージョンとして、BeagleBone Black用のイメージを公開しました。IoTとUbuntuとDockerとNinja SphereとBeagleBone Blackとワクワクがとまらないので、SDカードに焼いてBeagleBone Blackで動かしてみます。Smart things powered by snappy Ubuntu Core on ARM and x86 Ubuntu CoreのIoT用バージョンをCanonicalがローンチ
---

CanonicalがSnappy Ubuntu Coreの[IoTバージョン](http://www.ubuntu.com/things)として、BeagleBone Black用のイメージを[公開](https://developer.ubuntu.com/en/snappy/porting/)しました。IoTとUbuntuとDockerとNinja SphereとBeagleBone Blackとワクワクがとまらないので、SDカードに焼いてBeagleBone Blackで動かしてみます。

* [Ubuntu CoreのIoT用バージョンをCanonicalがローンチ](http://jp.techcrunch.com/2015/01/21/20150120ubuntu-of-things/)
* [Smart things powered by snappy Ubuntu Core on ARM and x86](http://www.markshuttleworth.com/archives/1445)

<!-- more -->

## Ninja Sphere

[Ninja Shield for BeagleBone Black](http://shop.ninjablocks.com/collections/ninja-blocks/products/ninja-shield-for-beaglebone-black)を買おうか悩んでいますが、[Ninja Sphere](https://ninjablocks.com/#home/)は、[Snappy Ubuntu Core](http://www.ubuntu.com/things)をベースにしているのでPRE-ORDERしてしまいそうです。Snappy appsを配布するアプリストアも用意しているようです。IoT用アプリの配布としてDockerイメージを使うのはおもしろいアイデアです。

## イメージのビルド

[Snappy for Devices porting guide](https://developer.ubuntu.com/en/snappy/porting/)の手順でセットアップしていきます。

Beaglebone Black用のイメージをビルドするため、Linux Mint 17 MATEにログインします。

``` bash
$ cat /etc/lsb-release
DISTRIB_ID=LinuxMint
DISTRIB_RELEASE=17
DISTRIB_CODENAME=qiana
DISTRIB_DESCRIPTION="Linux Mint 17 Qiana"
```

### ビルドの準備

Snappy用のPPAリポジトリの追加と必要なパッケージをインストールします。

``` bash
$ sudo add-apt-repository ppa:snappy-dev/beta
$ sudo apt-get update && sudo apt-get upgrade
$ sudo apt-get install snappy-tools bzr
```

Beaglebone Black向けに用意された、デバイス用のtarballをダウンロードします。

``` bash
$ cd ~/Downloads
$ wget -O ./device.tar.xz http://system-image.ubuntu.com/pool/device-bf0a49e75a3deb99855f186906f17d7668e2dcc3a0b38f5feade3e6f7c75e9b8.tar.xz
```

[Installing Ubuntu for devices](https://developer.ubuntu.com/en/start/ubuntu-for-devices/installing-ubuntu-for-devices/)を参考に、ubuntu-device-flashをインストールします。

``` bash
$ sudo apt-get install ubuntu-device-flash
```

### イメージのビルド

Beaglebone Black用のイメージをビルドします。

``` bash
$ sudo ubuntu-device-flash core \
    -o my-snappy.img \
    --size 4 \
    --channel ubuntu-core/devel \
    --device generic_armhf    \
    --platform am335x-boneblack \
    --enable-ssh \
    --device-part=./device.tar.xz
Fetching information from server...
Downloading and setting up...
67.31 MB / 67.31 MB [=========================================================] 100.00 % 63.68 KB/s
Running flashtool-asset commands
dd if=/tmp/device222703919/flashtool-assets/am335x-boneblack/MLO of=my-snappy.img count=1 seek=1 conv=notrunc bs=128k
0+1 レコード入力
0+1 レコード出力
79348 バイト (79 kB) コピーされました、 0.000348669 秒、 228 MB/秒

dd if=/tmp/device222703919/flashtool-assets/am335x-boneblack/u-boot.img of=my-snappy.img count=2 seek=1 conv=notrunc bs=384k
0+1 レコード入力
0+1 レコード出力
373048 バイト (373 kB) コピーされました、 0.000600729 秒、 621 MB/秒

New image complete
```

## イメージをSDカードに焼く

ビルドしたmy-snappy.imgをWindowsにSCP転送します。microSDカードは[SD Formatter](https://www.sdcard.org/jp/downloads/formatter_4/)を使い、論理サイズ調整をON、クイックフォーマットしておきます。

Cygwin64 Terminalを管理者として起動してmicroSDカードに焼きます。

``` bash
$ dd if=/cygdrive/c/Users/masato/Documents/my-snappy.img of=/dev/sdc bs=32M
119+1 レコード入力
119+1 レコード出力
4000000000 バイト (4.0 GB) コピーされました、 351.876 秒、 11.4 MB/秒
```

## Snappy Ubuntu Core on BeagleBone Black

BeagleBone BlackにSDカードを挿して起動します。電源供給のためUSBケーブルの1本はWindows7に挿します。もう1本はUSB-シリアル接続してコンソールを開きます。ユーザー名とパスワードは以下です。

* user: ubuntu
* passwd: ubuntu

![snappy-ubuntu-core.png](/2015/02/03/beagleboneblack-snappy-ubuntu-core/snappy-ubuntu-core.png)


### バージョン確認

Ubuntuのバージョンを確認します。Ubuntu Vivid Vervet 15.04がインストールされています。

``` bash
cat /etc/lsb-release
DISTRIB_ID=Ubuntu
DISTRIB_RELEASE=15.04
DISTRIB_CODENAME=vivid
DISTRIB_DESCRIPTION="Ubuntu Vivid Vervet (development branch)"
```

Snappyは0.1です。

``` bash
$ snappy -v
0.1
```

ubuntu-coreは、edgeの2がインストールされています。

``` bash
$ sudo snappy versions
Part         Tag   Installed  Available  Fingerprint     Active
ubuntu-core  edge  2          -          f442b1d8d6db3f  *
```

## ネットワーク設定


Dockerパッケージを検索しようとすると、ホスト名の名前解決に失敗します。

``` bash
$ snappy search docker
Traceback (most recent call last):
  File "/usr/bin/snappy", line 25, in <module>
    status = Main().__main__()
  File "/usr/lib/python3/dist-packages/snappy/main.py", line 195, in __main__
    return callback(args)
  File "/usr/lib/python3/dist-packages/snappy/main.py", line 401, in _do_search
    results = ClickDataSource().search(args.args)
  File "/usr/lib/python3/dist-packages/snappy/click.py", line 100, in search
    results = repo.search(",".join(terms))
  File "/usr/lib/python3/dist-packages/click/repository.py", line 141, in search
    resp, raw_content = http_request(url, headers=get_store_headers())
  File "/usr/lib/python3/dist-packages/click/network.py", line 70, in http_request
    curl.perform()
pycurl.error: (6, 'Could not resolve host: search.apps.ubuntu.com')
```

Windows7とはBeagleBone BlackはUSBケーブルで接続していますが、usb0のネットワークはみつかりません。

``` bash
$ ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: eth0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc pfifo_fast state DOWN group default qlen 1000
    link/ether 90:59:af:58:32:d1 brd ff:ff:ff:ff:ff:ff
```

### Ethernetケーブルをつなげる

eth0はDHCPのインタフェース設定が書いてありました。

```
$ cat /etc/network/interfaces.d/eth0
allow-hotplug eth0
iface eth0 inet dhcp
```

Ethernetケーブルを接続してネットワークを再起動すると、DHCPからIPアドレスが取得できました。

``` bash
$ sudo ifdown eth0
$ sudo ifup eth0
$ ip addr show eth0
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 90:59:af:58:32:d1 brd ff:ff:ff:ff:ff:ff
    inet 192.168.1.10/24 brd 192.168.1.255 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::9259:afff:fe58:32d1/64 scope link
       valid_lft forever preferred_lft forever
```

名前解決もできるようになりました。

``` bash
$ ping -c 3 www.yahoo.co.jp
PING www.g.yahoo.co.jp (124.83.203.233) 56(84) bytes of data.
64 bytes from f8.top.vip.ogk.yahoo.co.jp (124.83.203.233): icmp_seq=1 ttl=53 time=15.6 ms
```

## Snappyアプリを使う

[How to Install and Run Snappy Ubuntu Core on the BeagleBone]の動画をみながら使ってみます。

frameworksとappsはまだインストールされていません。

``` bash
$ snappy info
release: ubuntu-core/devel
frameworks:
apps:
```

アップデートできるバージョンは今のところありません。

``` bash
$ snappy update-versions
0 updated components are available with your current stability settings.
```

### WebDMのインストール

frameworksにWebDM(The web device manager)がインストールされていないので検索します。

``` bash
$ snappy search webdm
Part   Version  Description
webdm  0.1      WebDM
```

WebDMをインストールします。

``` bash
$ sudo snappy install webdm
webdm      5 MB     [================================================]    OK
Part   Tag   Installed  Available  Fingerprint     Active
webdm  edge  0.1        -          c94dd4609de5ba  *
```

WebDMのポートは4200のようです。

``` bash
$ sudo netstat -tlnp
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name
tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN      624/sshd
tcp6       0      0 :::4200                 :::*                    LISTEN      893/snappyd
tcp6       0      0 :::22                   :::*                    LISTEN      624/sshd
```

ブラウザを起動してWebDMの画面を表示します。

http://webdm.local:4200/

![snappy-store.png](/2015/02/03/beagleboneblack-snappy-ubuntu-core/snappy-store.png)

### Dockerが見つからない

Dockerをインストールしたかったのですが見つかりませんでした。

``` bash
$ sudo snappy install docker
No package 'docker' for ubuntu-core/devel
```
