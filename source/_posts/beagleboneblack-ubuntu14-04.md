title: "BeagleBone Blackのファームウェアを入れ直す - Part2: SDカードのUbuntu14.04.1へPuttyからシリアル接続する"
date: 2015-01-29 20:45:15
tags:
 - Ubuntu
 - BeagleBoneBlack
 - Putty
 - Linux-ARM
description: 前回、BeagleBone BlackのファームウェアをDebian 7.5に戻しました。この環境でSDカードからUbuntu 14.04.1を起動してみます。USBシリアル変換ケーブルでBBBとWindows7をつなぎ、Puttyからシリアル接続します。BBBのリビジョンA5BのSDカードからUbuntu 14.04.1を起動する場合、USBネットワーク接続がデフォルトで使えませんでした。
---

[前回](/2015/01/27/beagleboneblack-debian/)、BeagleBone BlackのファームウェアをDebian 7.5に戻しました。この環境でSDカードからUbuntu 14.04.1を起動してみます。USBシリアル変換ケーブルでBBBとWindows7をつなぎ、Puttyからシリアル接続します。BBBのリビジョンA5BのSDカードからUbuntu 14.04.1を起動する場合、USBネットワーク接続がデフォルトで使えませんでした。


<!-- more -->

## Ubuntu 14.04.1のイメージを用意

Cygwin64 Terminalを管理者として起動します。今回はイメージをeMMCにフラッシュせずにSDカードに焼いてから起動します。[Embedded Linux Wikiのページ](http://elinux.org/BeagleBoardUbuntu#BeagleBone.2FBeagleBone_Black)にある、[bone-ubuntu-14.04.1-console-armhf-2015-01-06-2gb.img.xz](https://rcn-ee.net/rootfs/2015-01-06/microsd/bone-ubuntu-14.04.1-console-armhf-2015-01-06-2gb.img.xz)をダウンロードします。


``` bash
$ cd /cygdrive/c/Users/masato/Downloads/
$ wget https://rcn-ee.net/rootfs/2015-01-06/microsd/bone-ubuntu-14.04.1-console-armhf-2015-01-06-2gb.img.xz
$ md5sum bone-ubuntu-14.04.1-console-armhf-2015-01-06-2gb.img,xz
9d602fdcaa350181c90bc90de15fd184 *bone-ubuntu-14.04.1-console-armhf-2015-01-06-2gb.img.xz
```

イメージをunxzコマンドで解凍して、SDカードに焼きます。

``` bash
$ unxz bone-ubuntu-14.04.1-console-armhf-2015-01-06-2gb.img.xz
$ dd if=./bone-ubuntu-14.04.1-console-armhf-2015-01-06-2gb.img of=/dev/sdc
```

## USB-シリアル変換ケーブルのピンを刺す

Prolific社製のPL-2303チップセットの採用している[USB-シリアル変換ケーブル](http://www.amazon.co.jp/dp/B00L8SP7U6)をアマゾンから650円で購入しました。4ピンケーブルのBlack、Green、Whiteの3本をBBBに刺します。

![usb-serial-4pin.png](/2015/01/29/beagleboneblack-ubuntu14-04/usb-serial-4pin.png)


## Puttyからシリアル接続

### Puttyの設定

USB-シリアル変換ケーブルをWindows7に接続すると自動でドライバがインストールされます。デバイスマネージャーでCOMポートを確認すると7を使っています。

![device-manager-com7.png](/2015/01/29/beagleboneblack-ubuntu14-04/device-manager-com7.png)

Puttyの新しいセッションの作成をします。

* 左カテゴリ > 接続 > シリアル

以下の値を入力して、セッション一覧に名前を付けて保存します。

* 接続先のシリアルポート: COM7
* 通信速度(ボー): 115200
* データ長(ビット): 8
* ストップビット: 1
* パリティ: なし
* フロー制御: なし

![putty-serical-port.png](/2015/01/29/beagleboneblack-ubuntu14-04/putty-serial-port.png)

### USB-シリアル接続

最初に電源供給用にBeagleBone Blackのmini USBケーブルを[USB急速充電器](http://www.amazon.co.jp/dp/B00GTGETFG)などに接続して電源を入れます。その後に3ピンを刺したUSB-シリアル接続をWindows7に接続します。

![serial-ubuntu-14.04.png](/2015/01/29/beagleboneblack-ubuntu14-04/serial-ubuntu-14.04.png)

USB-シリアル接続を使い、PuttyからSDカードから起動しているUbuntu 14.04にログインすることができました。

