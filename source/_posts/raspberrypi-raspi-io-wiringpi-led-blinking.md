title: "Raspberry PiをWiringPiとJohnny-FiveのRaspi-ioを使ってLチカする"
date: 2015-03-31 23:30:11
tags:
 - RaspberryPi
 - WiringPi
 - Lチカ
 - Johnny-Five
 - Raspi-io
description: Johnny-FiveにはArdiono以外でもFirmata互換のインタフェースを使えるようにするIO Pluginsがあります。Raspberry Pi用のRaspi-ioを使ってLチカをしてみます。GPIOピンアサインが正しいことをWiringPiを使って確認します。
---

[Johnny-Five](https://github.com/rwaldron/johnny-five)にはArdiono以外でもFirmata互換のインタフェースを使えるようにする[IO Plugins](https://github.com/rwaldron/johnny-five/wiki/IO-Plugins)があります。Raspberry Pi用の[Raspi-io](https://www.npmjs.com/package/raspi-io)を使ってLチカをしてみます。GPIOピンアサインが正しいことを[WiringPi](http://wiringpi.com/)を使って確認します。

<!-- more -->

## ブレッドボード配線

Raspberry PiにLEDを配線します。アノードはGPIO17(P1-11)と接続します。抵抗器は1.0K ohmを使いました。

![raspi-led2.png](/2015/03/31/raspberrypi-raspi-io-wiringpi-led-blinking/raspi-led2.png)

## WiringPi

[WiringPi](http://wiringpi.com/)をインストールします。WiringPiを使うと`gpio`コマンドでRaspberry PiのGPIOを簡単に制御することができます。

WiringPiをビルドしてインストールします。

``` bash
$ git clone git://git.drogon.net/wiringPi
$ cd wiringPi
$ ./build
```

バージョンを確認します。

``` bash
$ gpio -v
gpio version: 2.26
Copyright (c) 2012-2015 Gordon Henderson
This is free software with ABSOLUTELY NO WARRANTY.
For details type: gpio -warranty
Raspberry Pi Details:
  Type: Model B, Revision: 2, Memory: 512MB, Maker: Egoman
```

### 使い方


GPIO17 を点灯します。

``` bash
$ gpio -g write 17 1
```

GPIO17 を消灯します。

``` bash
$ gpio -g write 17 1
```

## Raspi-io

プロジェクトを作成します。

``` bash
$ mkdir -p ~/node_apps/raspi-io-led
$ cd !$
```



package.jsonに必要なパッケージを記述します。

``` json  ~/node_apps/raspi-io-led/package.json
{
    "name": "raspi-io-led",
    "version": "0.0.1",
    "private": true,
    "dependencies": {
        "johnny-five":"0.8.50",
        "raspi-io": "3.1.0"
    },
    "scripts": {"start": "node app.js"}
}
```

`npm install`します。

``` bash
$ npm install
...
raspi-io@3.1.0 node_modules/raspi-io
├── raspi-board@2.1.1
├── raspi-gpio@1.2.1 (raspi-peripheral@1.1.1, nan@1.6.2)
├── raspi-pwm@1.3.1 (raspi-peripheral@1.1.1, nan@1.6.2)
├── raspi@1.3.0 (nan@1.6.2, raspi-wiringpi@1.0.4)
└── raspi-i2c@1.0.0 (ini-builder@1.0.3, raspi-peripheral@1.1.1, execSync@1.0.2, i2c-bus@0.11.1)
```

app.jsを書きます。P1-11(GPIO17)をLチカします。

```js ~/node_apps/raspi-io-led/app.js
var raspi = require('raspi-io');
var five = require('johnny-five');
var board = new five.Board({
  io: new raspi()
});

board.on('ready', function() {
  (new five.Led('P1-11')).strobe();
});
```

LEDを操作する場合`sudo`が必要です。プログラムを実行するとLチカが始まります。

``` bash
$ sudo npm start

> raspi-io-led@0.0.1 start /home/pi/node_apps/raspi-io-led
> node app.js

1428244040441 Device(s) undefined
1428244040574 Connected RaspberryPi-IO
1428244040653 Repl Initialized
>>
```

 
