title: "FT232RLのUSB-TTLシリアル変換モジュールをESP826とRaspberry Piと使う"
date: 2015-04-10 11:44:00
categories:
 - IoT
tags:
 - FT232RL
 - ESP8266
 - RaspberryPi
description: これまでPL2303HXを搭載したUSB-TTLシリアル変換ケーブルを使っていました。電源供給が5V固定だったので3.3Vのモジュールを接続する場合はロジックレベルコンバータをはさんだり、Arduinoから3.3Vの電源を取ったりと配線が煩雑になってきました。正規品のFT232RLを搭載した3.3/5.0V切り換えスイッチ付きモジュールを安価に購入できたので配線してみます。
---

これまで[PL2303HXを搭載](http://www.amazon.co.jp/dp/B00L8SP7U6)したUSB-TTLシリアル変換ケーブルを使っていました。電源供給が5V固定だったので3.3Vのモジュールを接続する場合はロジックレベルコンバータをはさんだり、Arduinoから3.3Vの電源を取ったりと配線が煩雑になってきました。正規品のFT232RLを搭載した[3.3/5.0V切り換えスイッチ付き](http://www.aitendo.com/product/10149)モジュールを安価に購入できたので配線してみます。

<!-- more -->

## ESP8266

ブレッドボード配線図です。常に電源をONにする場合はCH_PDとVCCをはんだ付けしてしまえばブレッドボードも不要になります。

![FT232RL-ESP8366.png](/2015/04/10/ft232rl-esp8266-raspberry-pi/FT232RL-ESP8366.png)

* RXD (ESP8266) -> TXD (FT232RL)
* TXD (ESP8266) -> RXD (FT232RL)
* CH_PD (ESP8266) -> VCC (FT232RL)
* VCC (ESP8266) -> VCC (FT232RL)
* GND (ESP8266) -> GND (FT232RL)


ESP8266へはCoolTermから接続します。Options画面でシリアルポートとボーレートを設定します。ポートのxxxのところはデバイスのシリアル番号です。

* Port: usbserial-xxxxxxxx
* Baudrate: 9600


## Raspberry Pi

配線図です。Raspberry Piへの電源供給はFT232RLからでなく別途microUSBから行います。

![FT232RL-RasPi.png](/2015/04/10/ft232rl-esp8266-raspberry-pi/FT232RL-RasPi.png)

* GND (Raspberry Pi) -> GND (FT232RL)
* RXD (Raspberry Pi) -> TXD (FT232RL)
* TXD (Raspberry Pi) -> RXD (FT232RL)

Raspberry Piの場合はLinuxのシェルを使うためOSXのターミナルからscreenで接続します。

``` bash
$ screen /dev/tty.usbserial-xxxxxxxx 115200
```
