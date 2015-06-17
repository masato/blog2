title: OctalBoneScriptを使って3.14 kernelのBeagleBone BlackでLチカする
date: 2015-02-20 11:04:54
tags:
 - BeagleBoneBlack
 - BoneScript
 - OctalBoneScript
 - Lチカ
description: BeagleBone ScriptのSDカードからブートしているUbuntu 14.04でBoneScriptを使いLチカを試してみようと思ったのですがエラーが発生して動作しません。kernelを3.8.13から3.14に上げているのが問題になっているようです。OctalBoneScriptを使うと3.14でもLチカできそうなので試してみます。
---

BeagleBone ScriptのSDカードからブートしているUbuntu 14.04で[BoneScript](https://github.com/jadonk/bonescript)を使いLチカを試してみようと思ったのですがエラーが発生して動作しません。kernelを3.8.13から3.14に上げているのが問題になっているようです。[OctalBoneScript](https://github.com/theoctal/octalbonescript)を使うと3.14でもLチカできそうなので試してみます。

<!-- more -->

## GPIOのワイヤリング

ブレッドボートとのGPIOのワイヤリングは、[Blinking an LED with BeagleBone Black](https://learn.adafruit.com/blinking-an-led-with-beaglebone-black?view=all)を参考にしました。

![beaglebone_fritzing.png](/2015/02/20/bonescript-octalscript-blink-led/beaglebone_fritzing.png)

LEDや抵抗、ジャンパーワイヤーはAmazonで購入した[Arduinoをはじめようキット](http://www.amazon.co.jp/dp/B0025Y6C5G)の付属品を使います。

GPIOの拡張コネクタレイアウトは[こちら](http://beagleboard.org/support/bone101)を見ながら作業します。P8とP9の位置は間違えないようにします。

![cape-headers.png](/2015/02/20/bonescript-octalscript-blink-led/cape-headers.png)

## BoneScriptのサンプル

[BoneScript](http://beagleboard.org/Support/bone101)はNode.jsでハードウェアをコントロールできるパッケージです。eMMCにインストールしたDebianのイメージでは最初から使える状態になっています。SDカードにインストールしたUbuntuにはBoneScriptはインストールされていないので、package.jsonに記述してnpmから[パッケージ](https://github.com/jadonk/bonescript)をインストールします。サンプルプログラムは[INSPIRE: BEAGLEBONE BLACK RESOURCES](http://inspire.logicsupply.com/p/workshop-materials.html)のワークショップを参考にコードを書きました。

```js app.js
/*eslint-env node */
var b = require('bonescript');  // Read library
 
var ledPin = "P8_10";  // Select pin
var state = 1;  // Set LED state
var interval = 200;  // Set interval (in ms)
 
b.pinMode(ledPin, b.OUTPUT);  // Set pin to output
 
function toggleLED() {  // Switch LED on or off
   state = state ? 0 : 1;
   b.digitalWrite(ledPin, state);
}
setInterval(toggleLED, interval);  // Call function at set interval
```

BoneScriptのバージョンは0.2.4です。

``` bash
$ npm list bonesctipt
cylon-test@0.0.1 /home/ubuntu/node_modules/orion/.workspace/led
└── bonescript@0.2.4  extraneous
```

### pinModeのエラー

プログラムを実行するとGPIOを操作するpinModeメソッドでエラーが発生してしまいます。どうもカーネルのバージョンが変わるとデバイス名に変更があるようです。BoneScriptは3.8.xのカーネルでないとpinModeのメソッドは動作しないようです。

``` bash
$ sudo node app.js
...
/home/ubuntu/node_modules/orion/.workspace/led/node_modules/bonescript/index.js:161
    if(typeof resp.err != 'undefined') {
                  ^
TypeError: Cannot read property 'err' of undefined
    at Object.f.pinMode (/home/ubuntu/node_modules/orion/.workspace/led/node_modules/bonescript/index.js:161:19)
```

SDカードのUbuntu14.04はカーネルを3.14に上げていました。

``` bash
$ uname -a
Linux arm 3.14.31-ti-r49 #1 SMP PREEMPT Sat Jan 31 14:17:42 UTC 2015 armv7l armv7l armv7l GNU/Linux
```

[Problem with GPIO / pinMode #93](https://github.com/jadonk/bonescript/issues/93)を読むと[OctalScript](https://github.com/theoctal/octalbonescript)を推奨しているのでこちらを試してみます。

### 3.8.x kernelでないとCylon.jsもエラーになる

だだしこの状態だとCylon.jsの[cylon-beaglebone](https://github.com/hybridgroup/cylon-beaglebone)もLEDのデバイスを操作できません。Cylon.jsを動かすことが目標なので不都合がでてきます。しばらくはCylon.jsを使う場合、kernelを3.8.13のままにしているeMMCのDebianをブートします。

## OctalBoneScriptのサンプル

[OctalScript](https://github.com/theoctal/octalbonescript)はBoneScriptをより安定させ良くテストされたNode.jsのライブラリを提供することを目的としています。そのためBoneScriptで書いたコードはOctalBoneScriptと互換性があります。前回コードから`require`するモジュールをoctalbonescriptに変更します。また秒でLチカが終了するようにタイマーのクリアも追加しました。

``` js app.js
/*eslint-env node */
var b = require('octalbonescript');  // Read library
 
var ledPin = "P8_10";  // Select pin
var state = 1;  // Set LED state
var interval = 200;  // Set interval (in ms)
 
b.pinMode(ledPin, b.OUTPUT);  // Set pin to output
 
function toggleLED() {  // Switch LED on or off
   state = state ? 0 : 1;
   b.digitalWrite(ledPin, state);
}
 
var timer = setInterval(toggleLED, interval);  // Call function at set interval

setTimeout(function(){ clearInterval(timer)},3000);
```


OctalBoneScriptのバージョンは0.4.12です。

``` bash
$ npm list octalbonescript
cylon-test@0.0.1 /home/ubuntu/node_modules/orion/.workspace/led
└── octalbonescript@0.4.12 
```

`DEBUG=bone`を追加して起動するとデバッグモードで起動できます。ようやくLチカができるようになりました。

```
$ sudo DEBUG=bone node app.js 
debug: is_ocp() = /sys/devices/ocp.3
debug: is_cape_universal() = /sys/devices/ocp.3/cape-universal.61
debug: Using Universal Cape interface
debug: Enabling analog inputs
debug: load_dt_sync(cape-bone-iio,,)
debug: onFindCapeMgr: path = undefined
error: CapeMgr not found: undefined
debug: load_dt resp: {"err":"CapeMgr not found: undefined"}
debug: load_dt return: false
debug: index.js loaded
debug: pinMode(P8_10,out);
debug: hw.setPinMode(P8_10,gpio,{"value":true});
debug: returned from setPinMode
debug: hw.exportGPIOControls(P8_10,out,[object Object]);
debug: gpio: 68 already exported.
debug: digitalWrite(P8_10,0);
debug: gpioFile = /sys/class/gpio/gpio68/value
debug: digitalWrite(P8_10,1);
debug: gpioFile = /sys/class/gpio/gpio68/value
debug: digitalWrite(P8_10,0);
debug: gpioFile = /sys/class/gpio/gpio68/value
debug: digitalWrite(P8_10,1);
```

### error: CapeMgr not found: undefined

Lチカは成功しましたが`error: CapeMgr not found: undefined`のエラーメッセージが出ています。すでにリポジトリのIssuesに上がっていました。[error: CapeMgr not found: undefined #11](https://github.com/theoctal/octalbonescript/issues/11)

kernel 3.14ではCapeの構成を動的に変更することができなくなっているようです。OctacBoneScriptではGPIOとしてPinを使うモード変更はサポートされていないためエラーが発生しますが、一応Lチカはできるみたいです。
