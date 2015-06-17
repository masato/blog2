title: "Intel Edisonをスイッチサイエンス版Eagletに挿してCylon.jsでLチカする"
date: 2015-03-09 23:13:57
tags:
 - IntelEdison
 - Cylonjs
 - Nodejs
 - Eaglet
 - Lチカ
 - Grove
description: Edison用の拡張基板にはMini BreakoutことIntel Edison Breakout Board Kitを使っていました。Lチカをしようと調べていると直接LEDをつないではいけないようです。EdisonのGPIOの電圧はArduinoのように5Vや3.3Vではなく1.8Vしかありません。この電圧レベルの信号ではLEDを光らせることはできないのでI2C通信用レベル変換をする必要がありますが、毎回変換モジュールを経由するのは面倒です。スイッチサイエンス版Eaglet (MFTバージョン)はArdiono拡張基板と違いFRISKサイズを維持しながら3.3Vにレベル変換してくれます。Groveコネクタが付いているのもうれしいです。
---

Edison用の拡張基板にはMini Breakoutこと[Intel Edison Breakout Board Kit](https://www.switch-science.com/catalog/1957/)を使っていました。Lチカをしようと調べていると直接LEDをつないではいけないようです。EdisonのGPIOの電圧はArduinoのように5Vや3.3Vではなく1.8Vしかありません。この電圧レベルの信号ではLEDを光らせることはできないのでI2C通信用レベル変換をする必要がありますが、毎回変換モジュールを経由するのは面倒です。[スイッチサイエンス版Eaglet (MFTバージョン)](https://www.switch-science.com/catalog/2070/)はArdiono拡張基板と違いFRISKサイズを維持しながら3.3Vにレベル変換してくれます。Groveコネクタが付いているのもうれしいです。

<!-- more -->

## SSH接続

[前回](/2015/03/06/intel-edison-setting-up/)Edisonには無線LANの設定をしています。EdisonをEagletに挿してmicroUSBポートをUSB電源アダプタにつなぐだけでSSH接続できるようになります。DHCPから以下のIPアドレスが振られています。


``` bash
$ ssh root@192.168.1.14
```

## Cylon.js

Edisonアダプタのnpmパッケージは[cylon-intel-iot](https://github.com/hybridgroup/cylon-intel-iot)になります。

### コーディング

プロジェクトを作成します。

``` bash
$ mkdir -p ~/node_apps/led-blinking
$ cd !$
```

package.jsonにCylon.jsのEdison用アダプターを追加します。
 
``` json ~/node_apps/led-blinking/package.json
{
  "name": "led-blinking",
  "version": "0.0.1",
  "private": true,
  "dependencies": {
    "cylon-intel-iot": "0.5.0"
  },
  "scripts": {"start": "node app.js"}
}
```

パッケージをインストールします。

``` bash
$ npm install
```

Node.jsのプログラムは[Cylon.js](http://cylonjs.com/documentation/platforms/edison/)のサンプルを使いますが、Eaglet用にピン番号を変更します。スイッチサイエンスの[C++でLED点滅](http://trac.switch-science.com/wiki/IntelEdisonBlinkCPlusPlus)のページにあるC++のコードを参考にすると、mraa番号は37を使えばよいみたいです。
 
``` js ~/node_apps/led-blinking/app.js
var Cylon = require('cylon');

Cylon.robot({
  connections: {
    edison: { adaptor: 'intel-iot' }
  },

  devices: {
    led: { driver: 'led', pin: 37 }
  },

  work: function(my) {
    every((1).second(), my.led.toggle);
  }
}).start();
```

### 電源ランプ兼D13をLチカ

プログラムを実行してLチカさせます。

``` bash
$ npm start

> led-blinking@0.0.1 start /home/root/node_apps/led-blinking
> node app.js

I, [2015-03-09T14:39:13.895Z]  INFO -- : [Robot 56633] - Initializing connections.
I, [2015-03-09T14:39:14.897Z]  INFO -- : [Robot 56633] - Initializing devices.
I, [2015-03-09T14:39:14.916Z]  INFO -- : [Robot 56633] - Starting connections.
I, [2015-03-09T14:39:14.924Z]  INFO -- : [Robot 56633] - Starting devices.
I, [2015-03-09T14:39:14.926Z]  INFO -- : [Robot 56633] - Working.
```