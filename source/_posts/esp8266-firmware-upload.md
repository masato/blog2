title: "ESP8266のファームウェアを更新する"
date: 2015-04-03 15:26:57
categories:
 - IoT
tags:
 - ESP8266
 - OSX
 - CoolTerm
 - Arduino
description: 前回はESP8266 ATファームウェアを使いWi-Fiの接続を確認しました。ATコマンドを使って接続処理を書くのは面倒なのでなにかよい方法を探しています。あとでファームウェアを更新することになるので、ATコマンドが使えるカスタムファームウェアを使い手順を確認することにします。以下のサイトを参考にしました。
---

[前回](/2015/03/29/arduino-esp8266-usb-ttl-serial/)はESP8266 ATファームウェアを使いWi-Fiの接続を確認しました。ATコマンドを使って接続処理を書くのは面倒なのでなにかよい方法を探しています。あとでファームウェアを更新することになるので、ATコマンドが使えるカスタムファームウェアを使い手順を確認することにします。以下のサイトを参考にしました。

* [Getting started with the esp8266 and Arduino](http://www.madebymarket.com/blog/dev/getting-started-with-esp8266.html)
* [ESP8266Firmware](http://www.electrodragon.com/w/ESP8266_Firmware)
* [How to update the ESP8266 Module's firmware](https://developer.mbed.org/users/sschocke/code/WiFiLamp/wiki/Updating-ESP8266-Firmware)

<!-- more -->

## ブレッドボード配線

Arduino Unoは3.3Vの電源供給に使います。ファームウェアのアップロードをする場合は[前回](/2015/03/29/arduino-esp8266-usb-ttl-serial/)の配線に加えて、ESP8266のGPIO0をGNDに接続します。ESP8266と接続したUSB-TTLシリアル変換ケーブルをOSXに接続し、Arduino UnoのUSBケーブルは電源アダプタにつなぎます。

* RX (ESP8266) -> TX (USB-TTL)
* TX (ESP8266) -> RX (USB-TTL)
* CH_PD (ESP8266) -> VCC (Arduino Uno 3.3V)
* VCC (ESP8266) -> VCC (Arduino Uno 3.3V)
* GND (ESP8266) -> GND (Arduino)
* GND (USB-TTL) -> GND (Arduino)
* GPIO0 (ESP8266) -> GND (Arduino)

![ESP8266-firm.png](/2015/04/03/esp8266-firmware-upload/ESP8266-firm.png)

## esptool

[esptool](https://github.com/themadinventor/esptool/)を使ってファームウェアをESP8266に書き込みます。システムワイドにインストールします。

``` bash
$ cd ~/arduino_apps
$ git clone https://github.com/themadinventor/esptool/
$ cd esptool
$ sudo python setup.py install
...
Installed /Library/Python/2.7/site-packages/pyserial-2.7-py2.7.egg
Finished processing dependencies for esptool==0.1.0
$ which esptool.py
/usr/local/bin/esptool.py
```

## ファームウェアダウンロード

ATコマンドでファームウェア更新の確認をしたいので、[Electricdragon](http://www.electrodragon.com/)の[Customized AT-thinker Firmware](http://www.electrodragon.com/w/ESP8266_Firmware)を使います。[Googleドライブ](https://drive.google.com/folderview?id=0B_ctPy0pJuW6d1FqM1lvSkJmNU0&usp=sharing)から`AI-v0.9.5.0 AT Firmware.bin`を`esp8266.9.5.0.bin`にファイル名を変更してOSXに保存します。


## ファームウェアの書き込み

esptool.pyを使ってファームウェアを書き込みます。

``` bash
$ esptool.py -p /dev/tty.usbserial write_flash 0x000000 esp8266.9.5.0.bin
Connecting...
Erasing flash...
Writing at 0x0007ec00... (100 %)

Leaving...
```

書き込みが終了したら、GPIO0とGNDの接続を外してESP8266の電源を入れ直します。OSXの[CoolTerm](http://freeware.the-meiers.org/)を起動してATコマンドのテストをします。ファームウェアのバージョンは`0.9.5`に上がりました。

```
AT+GMR
00200.9.5(b1)
compiled @ Dec 25 2014 21:40:28
AI-THINKER Dec 25 2014

OK
```

![firmware-0950.png](/2015/04/03/esp8266-firmware-upload/firmware-0950.png)