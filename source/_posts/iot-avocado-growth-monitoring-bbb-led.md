title: "IoTでアボカドの発芽促進とカビを防止する - Part2: BeagleBone Blackで植物育成LEDライトを制御する"
date: 2015-07-05 09:24:25
categories:
 - IoT
tags:
 - IoT
 - BeagleBoneBlack
 - LED
 - アボカド
description: アボカドの種の水耕栽培を始めて１週間くらい経ちました。芽がでるまで1ヶ月くらいかかるようです。植物育成LEDライトはカビの防止と生育に効果があるみたいなので、半日照明を当てていたら乾燥しすぎて種にヒビが入ってしまいました。温度や湿度を見ながらLEDライトを当てる時間を調整する必要があります。LEDは電源コンセントにつながる普通のライトです。マイコンから制御する方法をいろいろ考えていると、USB連動電源タップとプログラムから制御可能なUSBハブを使うと上手くいきそうです。以下のポストを参考にさせていただきました。
---

アボカドの種の水耕栽培を始めて１週間くらい経ちました。芽がでるまで1ヶ月くらいかかるようです。植物育成LEDライトはカビの防止と生育に効果があるみたいなので、半日照明を当てていたら乾燥しすぎて種にヒビが入ってしまいました。温度や湿度を見ながらLEDライトを当てる時間を調整する必要があります。

電源コンセントにつなげる普通のLEDライトをマイコンから制御する方法を調べました。USB連動電源タップとプログラムから制御可能なUSBハブを使うと上手くいきそうです。以下のポストを参考にさせていただきました。


* [USBポートの電源を制御する_その１](http://www.ebimemo.net/diary/?date=20110530)
* [スゴイHUB （SUGOI HUB4X）は本当に凄かった - USBポートの電源制御 -](http://solarisintel.blog.fc2.com/blog-entry-5.html)
* [モーション検出でUSBライトのON・OFF](http://yamada468.blogspot.jp/2012/05/usbonoff.html)

<!-- more -->

## 用意するもの

以下の部材を用意しました。

* [BeagleBone Black](http://www.amazon.co.jp/dp/B00CHHMZ9S)
* [SUGOI HUB4Xシリーズ](http://www.amazon.co.jp/dp/B001Q6N4EQ)
* [赤/青 植物育成LEDライト E26 小型スポットライト](http://www.amazon.co.jp/dp/B00LTKSI5O)
* [サンワサプライ TAP-RE7U USB連動タップ](http://www.amazon.co.jp/dp/B00008KEJ0)
* [PLANEX 無線LAN子機](http://www.amazon.co.jp/dp/B00ESA34GA)

[WeMo](http://www.belkin.com/us/Products/home-automation/c/wemo-home-automation/)をもっと単純にした感じです。

## Raspberry Pi 2はUSBの電源を上手く制御できない

[Raspberry Pi B+ turn usb power off](https://www.raspberrypi.org/forums/viewtopic.php?f=44&t=11558)にもポストがあります。手元にあるRaspberry Pi 2だとUSBポートの電源を上手く制御できませんでした。一応プログラムからON/OFFできるですが、ハブにつながったUSBが全て同時に制御されます。USBには無線LANのドングルも挿しているのでネットワークが切断してしまい困ります。

Raspberry Pi 2の場合、4つあるポートのどこにUSB連動電源タップを接続してもPort 2しかON/OFFの制御ができませんでした。電源はすべてのポートで連動しているようです。

```bash
$ hub-ctrl -h 0 -P 2 -p 0
```

## AC Power Control by USB Hub

[AC Power Control by USB Hub](http://www.gniibe.org/development/ac-power-control-by-USB-hub/index.html)に紹介されっているライブラリを使うと、対応しているUSBハブのポートの電源を制御することができます。Cで実装された[hub-ctrl.c](http://www.gniibe.org/oitoite/ac-power-control-by-USB-hub/hub-ctrl.c)と、Pythonの[hub_ctrl.py](http://git.gniibe.org/gitweb/?p=gnuk/gnuk.git;a=blob;f=tool/hub_ctrl.py)があります。


### Cのソースコード

BeagleBone BlackにはUbuntu 14.04をインストールしています。

```bash
$ cat /etc/os-release
NAME="Ubuntu"
VERSION="14.04.1 LTS, Trusty Tahr"
$ cat /proc/version
Linux version 3.14.31-ti-r49 (root@a6-imx6q-wandboard-2gb) (gcc version 4.8.2 (Ubuntu/Linaro 4.8.2-19ubuntu1) ) #1 SMP PREEMPT Sat Jan 31 14:17:42 UTC 2015
```

プログラムの実行に必要なパッケージをインストールします。

```bash
$ sudo apt-get update
$ sudo apt-get install libusb-dev
```

Cのソースコードをダウンロードしてコンパイルします。

```bash
$ mkdir ~/hub
$ cd !$
$ wget http://www.gniibe.org/oitoite/ac-power-control-by-USB-hub/hub-ctrl.c
$ gcc -O2 hub-ctrl.c -o hub-ctrl -lusb
```

コンパイルしたバイナリのヘルプを確認します。

```bash
$ ./hub-ctrl --help
Usage: ./hub-ctrl [{-h HUBNUM | -b BUSNUM -d DEVNUM}] \
          [-P PORT] [{-p [VALUE]|-l [VALUE]}]
```

### Pythonのソースコード

Pythonの[hub_ctrl.py](http://git.gniibe.org/gitweb/?p=gnuk/gnuk.git;a=blob;f=tool/hub_ctrl.py)をダウンロードして実行権限を付けます。

```bash
$ wget -O hub_ctrl.py "http://git.gniibe.org/gitweb/?p=gnuk/gnuk.git;a=blob_plain;f=tool/hub_ctrl.py;hb=HEAD"
$ chmod u+x hub_ctrl.py
```

Pプログラム実行に必要なライブラリをインストールします。今回の環境では[pyusb](http://walac.github.io/pyusb/)は`--pre`フラグが必要でした。

```bash
$ sudo apt-get install git build-essential python-dev python-smbus
$ sudo pip install --pre pyusb
...
Successfully installed pyusb-1.0.0b2
```

ヘルプを確認します。

```bash
$ sudo ./hub_ctrl.py -h
Usage: ./hub_ctrl.py [{-h HUBNUM | -b BUSNUM -d DEVNUM}]
          [-P PORT] [{-p [VALUE]|-l [VALUE]}]
```

## 使い方

[USBポートの電源を制御する_その１](http://www.ebimemo.net/diary/?date=20110530)を参考にさせていただきます。

普通の電源タップにUSBケーブルが繋がっています。USBケーブルをテレビやパソコンなどに接続して、テレビの電源と連動してハードディスクなど周辺機器の電源もON/OFFさせます。節電が主な用途らしいです。

今回はこれを応用して、植物育成LEDライトを連動する機器にします。LEDライトのコンセントUSB連動電源タップに挿してスイッチをONの状態にしておきます。電源タップのUSBケーブルはプログラムから電源の制御が可能なUSBハブのポートに接続してこれと連動させます。

BeagleBone BlackのPythonのプログラムから、るUSBハブのポートの電源をON/OFFさせることで、LEDライトの電源をコントールしてみようと思います。

### lsusbコマンド

`lsusb`コマンドを実行して、電源をON/OFFさせるUSBハブのポート番号、Bus番号、Device番号を確認します。

```bash
$ lsusb
Bus 001 Device 003: ID 0411:01a2 BUFFALO INC. (formerly MelCo., Inc.) WLI-UC-GNM Wireless LAN Adapter [Ralink RT8070]
Bus 001 Device 002: ID 0409:005a NEC Corp. HighSpeed Hub
Bus 001 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub Wireless LAN Adapter [Ralink RT8070]
```

[AC Power Control by USB Hub](http://www.gniibe.org/development/ac-power-control-by-USB-hub/index.html)のページに対応USBハブの記載がありますが、NEC製の`Bus 001 Device 002`がUSB連動電源タップをつないでいるデバイスです。root権限で`lsusb`コマンドの詳細が表示できます。

```bash
$ sudo lsudb -v
...
Bus 001 Device 002: ID 0409:005a NEC Corp. HighSpeed Hub
Device Descriptor:
  bLength                18
  bDescriptorType         1
  bcdUSB               2.00
  bDeviceClass            9 Hub
...
 Hub Port Status:
   Port 1: 0000.0100 power
   Port 2: 0000.0100 power
   Port 3: 0000.0100 power
   Port 4: 0000.0503 highspeed power enable connect
```

`Port 4`には無線LANのドングルを挿しています。BeagleBone BlackがクラウドのMQTTブローカーに接続して、LEDライトをコントールするメッセージをsubscribeするために使います。USBハブは`Port 1`が電源制御可能なポートになっていました。

### Cのコマンド

Cのコマンドのフラグの指定方法です。バスとデバイス番号の組み合わせか、ハブ番号を指定します。

``` bash
# hub-ctrl -b バス番号 -d デバイス番号 -P ポート番号 -p 0 または 1
# hub-ctrl -h ハブ番号 -P ポート番号 -p 0 または 1
```

LEDライトを点灯する場合は、`-p`フラグを1にします。

```bash
$ sudo ./hub-ctrl -b 1 -d 2 -P 1 -p 1
```

消灯する場合は0を指定します。点灯はすぐ連動しますが、消灯の方はコマンドを実行してから５秒くらいかかります。

```bash
$ sudo ./hub-ctrl -b 1 -d 2 -P 1 -p 0
```

### Pythonのコマンド

Pythonのコマンドの場合は、バスとデバイス番号を指定するとエラーになりました。

```bash
$ sudo ./hub_ctrl.py -b 1 -d 2 -P 1 -p 1
Traceback (most recent call last):
  File "./hub_ctrl.py", line 236, in <module>
    uh = dev_hub.open()
NameError: name 'dev_hub' is not defined
```

ハブの番号の指定で動作しました。`-p 1`で点灯します。

```bash
$ sudo ./hub_ctrl.py -h 0 -P 1 -p 1
```

0を指定すると消灯します。

```bash
$ sudo ./hub_ctrl.py -h 0 -P 1 -p 1
```

## アボカドとコネクテッドデバイス

赤と青のLEDライトが妖しげです。アボカドの種は３個育てていますが、1つは元気がなく、1つはカビ防止のつもりがLEDの当てすぎで乾燥して割れてしまい、1つだけ順調に育っています。

Raspberry Pi 2はBME280の環境センサーのデータをクラウドのMQTTブローカーにpublishしています。BeagleBlackはクラウドのMQTTブローカーをsubscribeしてLEDライトのコントロールに使っています。

![avocado_bbb.JPG](/2015/07/05/iot-avocado-growth-monitoring-bbb-led/avocado_bbb.JPG)