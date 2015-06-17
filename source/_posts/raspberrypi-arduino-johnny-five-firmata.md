title: "Raspberry PiからArduinoをJohnny-FiveとFirmataで操作する"
date: 2015-03-22 10:22:30
tags:
 - RaspberryPi
 - Arduino
 - ArduinoUno
 - Gort
 - Firmata
 - Johnny-Five
 - Nodejs
description: これまでOSXやChromebookをホストマシンとして、Arduino FirmataをCylon.jsやJohnny-Fiveから操作していました。ロボットやRCカーを作ろうとするといつまでも有線では困ります。センサーやアクチュエーターなどのハードウェア操作はArduinoは優れています。一方ArduinoへFirmataプロトコルのコマンドを発行したり、外部からWi-Fi経由でMQTTやWebSocketのメッセージを取得する場合はLinuxが動作するRaspberry PiやEdisonが便利です。Arduino YunやLinino OneのようにArduinoとLinuxの両方をワンボードに乗せたマイコンもあります。impやSparkのようにクラウドからデバイスに対して直接コマンドを発行できると、これまでのクラウドの技術を活用してコネクテッドデバイスの可能性が広がります。
---

これまでOSXやChromebookをホストマシンとして、[Arduino Firmata](http://arduino.cc/en/reference/firmata)を[Cylon.js](http://cylonjs.com/)や[Johnny-Five](https://github.com/rwaldron/johnny-five)から操作していました。ロボットやRCカーを作ろうとするといつまでも有線では困ります。センサーやアクチュエーターなどのハードウェア操作はArduinoは優れています。一方ArduinoへFirmataプロトコルのコマンドを発行したり、外部からWi-Fi経由でMQTTやWebSocketのメッセージを受信する場合はLinuxが動作するRaspberry PiやEdisonが便利です。[Arduino Yun](http://arduino.cc/en/Main/ArduinoBoardYun)や[Linino One](http://www.linino.org/)のようにArduinoとLinuxの両方をワンボードに乗せたマイコンもあります。[imp](https://electricimp.com/)や[Spark](https://www.spark.io/)のようにクラウドからデバイスに対して直接コマンドを発行できると、これまでのクラウドの技術を活用してコネクテッドデバイスの可能性が広がります。

<!-- more -->

## ホストマシン

ArduinoをFirmataプロトコルで操作するためのホストマシンはRaspberry Piになります。ディスプレイとキーボードを使う作業マシンはChromebookです。Raspberry PiとはUSBシリアル変換ケーブルで接続しています。Raspberry Piのデバイスファイルの`/dev/ttyUSB0`へscreenからシリアル接続します。

``` bash
$ sudo enter-chroot
$ sudo screen /dev/ttyUSB0 115200
```

Raspberry Piは電源が弱いので、USBシリアル変換ケーブルのGPIOとmicroUSBの両方から電源供給します。

![raspberrypi-arduino.jpg](/2015/03/22/raspberrypi-arduino-johnny-five-firmata/raspberrypi-arduino.jpg)


## Firmata

### Gortのインストール

Raspberry PiからArduinoをFirmataプロトコルで操作するために、[Gort](http://gort.io/)をインストールします。

``` bash
$ wget https://s3.amazonaws.com/gort-io/0.3.0/gort_0.3.0_linux_arm.tar.gz
$ tar zxvf gort_0.3.0_linux_arm.tar.gz 
gort_0.3.0_linux_arm/gort
gort_0.3.0_linux_arm/README.md
gort_0.3.0_linux_arm/LICENSE
$ sudo cp gort_0.3.0_linux_arm/gort /usr/local/bin
$ gort --version
gort version 0.3.0
```

### Firmataファームウェアを書き込む

Raspberry PiとArduinoをUSBケーブルで接続します。シリアルポートをスキャンするとArduinoのデバイスファイルは`/dev/ttyACM0`になっています。

```
$ gort scan serial

1 serial port(s) found.

1. [/dev/ttyACM0] - [usb-Arduino__www.arduino.cc__0043_55431313338351E0F111-if00]
  USB device:  Bus 001 Device 006: ID 2341:0043 Arduino SA Uno R3 (CDC ACM)
```

Arduinoのデバイスファイルにpiユーザーが書き込めるように`dialout`グループに追加します。

``` bash
$ ls -l /dev/ttyACM0
crw-rw---T 1 root dialout 166, 0 Mar 23 16:51 /dev/ttyACM0
$ sudo usermod -aG dialout pi
```

Raspberry PiからArduinoへFirmataファームウェアを書き込むために、gortコマンドを使い[AVRDUDE](http://www.nongnu.org/avrdude/)をインストールします。

``` bash
$ gort arduino install
Attempting to install avrdude with apt-get.
...
```

Raspberry Piからgortとavrdudeを使いArduinoにFirmataファームウェアを書き込みます。

``` bash
$ gort arduino upload firmata /dev/ttyACM0
avrdude: AVR device initialized and ready to accept instructions
...
avrdude done.  Thank you.
```


## Johnny-Fiveのサンプル


### Node.js

Rasbperry PiにNode.jsをインストールします。今回は[node-arm](http://node-arm.herokuapp.com/)から最新のdebパッケージをダウンロードして使います。`v0.12.0`がインストールされました。

``` bash
$ wget http://node-arm.herokuapp.com/node_latest_armhf.deb
$ sudo dpkg -i node_latest_armhf.deb
$ node -v
v0.12.0
$ npm -v
2.5.1
```

### Lチカ

Johnny-Fiveを使ってRasbperry PiからArduinoを操作します。いつものようにLチカのサンプルです。まずプロジェクトを作成します。

``` bash
$ mkdir -p ~/node_apps/johnny-five-start
$ cd !$
```

package.jsonにJohnny-Fiveのパッケージを追加します。

``` json ~/node_apps/johnny-five-start/package.json
{
  "name": "johnny-five-start",
  "version": "0.0.1",
  "private": true,
  "dependencies": {
    "johnny-five": "0.8.49"
  },
  "scripts": {"start": "node app.js"}
}
```

簡単なプログラムを用意します。1秒間隔でLチカを繰り返します。

``` js ~/node_apps/johnny-five-start/app.js
var five = require("johnny-five");
var board = new five.Board();
board.on("ready", function() {
  var led = new five.Led(13);
  led.blink(1000);
});
```

`npm start`でapp.jsを実行するとLチカが開始します。プログラムを終了するときはCtrl+Cを2回押します。

``` bash
$ npm install
$ npm start
> node app.js

1427098288985 Device(s) /dev/ttyACM0 
1427098289243 Connected /dev/ttyACM0 
1427098292815 Repl Initialized 
>> 
```


