title: "Johnny-FiveでJavaScript Roboticsをはじめる"
date: 2015-03-20 23:09:46
tags:
 - JavaScript
 - Nodejs
 - Robotics
 - Cylonjs
 - Arduino
 - Firmata
 - Gort
 - Lチカ
description: EspruinoやTessel、imp、Spark、EdisonのmraaなどNode.js/JavaScriptのインタフェースを持ったマイコンや小型コンピューターが増えてきました。これまでIoTのJavaScriptフレームワークとしてCylon.jsを主に使っていました。Cylon.jsのDSLはシンプルで良いのですが、もうちょっとNode.jsらしく書けないか別の方法を探しています。mraaはLow Level過ぎたりなかなか難しいです。Johnny-Fiveも同様のJavaScript Roboticsフレームワークです。FirmataプロトコルがベースになっているのでArduinoを使ってLチカのサンプルを書いてみます。
---

[Espruino](http://www.espruino.com/)や[Tessel](https://tessel.io/)、[imp](https://electricimp.com/)、[Spark](https://www.spark.io/)、Edisonの[mraa](https://github.com/intel-iot-devkit/mraa)などNode.js/JavaScriptのインタフェースを持ったマイコンや小型コンピューターが増えてきました。これまでIoTのJavaScriptフレームワークとして[Cylon.js](http://cylonjs.com/)を主に使っていました。Cylon.jsのDSLはシンプルで良いのですが、もうちょっとNode.jsらしく書けないか別の方法を探しています。mraaはLow Level過ぎたりなかなか難しいです。[Johnny-Five](https://github.com/rwaldron/johnny-five)も同様のJavaScript Roboticsフレームワークです。FirmataプロトコルがベースになっているのでArduinoを使ってLチカのサンプルを書いてみます。


<!-- more -->

## FirmataとI/O plugin

Johnny-FiveはArduino以外のFirmataファームウェアを持たないマイコン用に、[I/O plugin](https://github.com/rwaldron/johnny-five/wiki/IO-Plugins)というFirmata互換のI/Oクラスを用意しています。インタフェースをFirmataにあわせてくれるので異なるマイコンにも同じようにプログラミングができます。Cylon.jsのDSLのアプローチよりJohnny-FiveのFirmataのI/Oクラスをベースにしたフレームワークの方が、ロジックをNode.jsで普通に実装できそうです。


## Johnny-Fiveのインストール

Johnny-FiveはArduino Firmataを使うので、[前回](/2015/03/15/cylonjs-arduino-uno-firmata-gort-osx/)OSXに構築したArduinoをCylon.jsからFirmataプロトコルで操作する環境がそのまま使えます。ざっとインストールコマンドだけ復習します。Arduino UnoとOSXをUSBシリアル接続します。[Gort](http://gort.io/)を使いavrdudeのインストールと、ArduinoにFirmataファームウェアをアップロードします。

``` bash
$ cd ~/Downloads
$ wget https://s3.amazonaws.com/gort-io/0.3.0/gort_0.3.0_darwin_amd64.zip
$ unzip gort_0.3.0_darwin_amd64.zip
$ sudo cp gort_0.3.0_darwin_amd64/gort /usr/local/bin/
$ gort arduino install
$ gort arduino upload firmata /dev/tty.usbmodem1421
```

Johnny-Fiveの実行環境としてNode.jsをインストールします。

``` bash
$ curl https://raw.githubusercontent.com/creationix/nvm/v0.24.0/install.sh | bash
$ nvm install 0.10
$ nvm use 0.10
v0.10.37
```

## Lチカ

リポジトリに[Examples](https://github.com/rwaldron/johnny-five/tree/master/docs)がたくさんあります。定番のLチカは[led-blink](https://github.com/rwaldron/johnny-five/blob/master/docs/led-blink.md)を使います。まずはプロジェクトを作成します。

``` bash
$ mkdir -p ~/node_apps/johnny-five-start
$ cd !$
```

package.jsonを用意します。johnny-fiveのパッケージを定義します。

```json ~/node_apps/johnny-five-start/package.json
{
  "name": "johnny-five-start",
  "version": "0.0.1",
  "private": true,
  "dependencies": {
    "johnny-five": "0.8.48"
  },
  "scripts": {"start": "node app.js"}
}
```

プログラムはとても単純です。1秒間隔でLチカを繰り返します。

```js ~/node_apps/johnny-five-start/app.js
var five = require("johnny-five");
var board = new five.Board();
board.on("ready", function() {
  var led = new five.Led(13);
  led.blink(1000);
});
```

`npm start`でプログラムを実行します。終了するときは`Ctrl+C`を2回押します。

``` bash
$ npm start
1426949491597 Device(s) /dev/cu.usbmodem1421
1426949491606 Connected /dev/cu.usbmodem1421
1426949494867 Repl Initialized
>>
(^C again to quit)
>>
1426949511237 Board Closing.
```

## REPL

Johnny-FiveはREPLが使えるのでインタラクティブにマイコンを操作できます。[REPL](https://github.com/rwaldron/johnny-five/blob/master/docs/repl.md)のサンプルを使います。Espruinoの[コンソール](/2015/03/18/espruino-quick-start/)もREPLとして使えました。JavaScriptを使うとコネクテッドデバイスの操作がイベント駆動で直感的に書けます。

```js ~/node_apps/johnny-five-start/repl.js
var five = require("johnny-five");
var board = new five.Board();
board.on("ready", function() {
  console.log("Ready event. Repl instance auto-initialized!");
  var led = new five.Led(13);
  this.repl.inject({
    on: function() {
      led.on();
    },
    off: function() {
      led.off();
    }
  });
});
```

プログラムを実行します。Returnを1回押してREPLの待ち受けを表示します。

``` bash
$ node repl.js
node repl.js
1426949978155 Device(s) /dev/cu.usbmodem1421
1426949978161 Connected /dev/cu.usbmodem1421
1426949981423 Repl Initialized
>> Ready event. Repl instance auto-initialized!

undefined
>>
```

プログラムではonとoffの関数を定義しているので、REPLからそれぞれ呼び出すことができます。`on()`でLEDを点灯し`off()`で消灯する命令をインタラクティブに実行することができます。

``` bash
>> on()
undefined
>> off()
undefined
>>
``` 
