title: "ArduinoのFirmataファームウェアをOSXのCylon.jsから使う"
date: 2015-03-15 11:28:00
tags:
 - Arduino
 - ArduinoUno
 - Cylonjs
 - Nodejs
 - Firmata
 - Gort
description: croutonのUbuntuにはNode.jsとCylon.jsをFirmataファームウェアを使って操作する環境を用意しました。OSX Yosemiteにも同じようにNod.jsの開発環境を用意します。FirmataのアップロードはArduino IDEを使わずにGortのCLIから行います。
---

[croutonのUbuntu](/2015/02/23/cylonjs-arduino-uno-firmata-gort/)にはNode.jsとCylon.jsをFirmataファームウェアを使って操作する環境を用意しました。OSX Yosemiteにも同じようにNod.jsの開発環境を用意します。FirmataのアップロードはArduino IDEを使わずに[Gort](http://gort.io/)のCLIから行います。

<!-- more -->

## OSXの環境構築

[Getting Started w/ Arduino on Mac OS X](http://www.arduino.cc/en/Guide/MacOSX)を読みながら環境構築をしていきます。

### FTDIのUSBシリアルドライバ

Arduino UnoとUSBシリアル接続するために、OSXにFTDIのUSBシリアルドライバをインストールします。FTDIの[VCP Drivers](http://www.ftdichip.com/Drivers/VCP.htm)のページから[OSX用のインストーラー](http://www.ftdichip.com/Drivers/VCP/MacOSX/FTDIUSBSerialDriver_v2_2_18.dmg)をダウンロードします。最新バージョンは`2.2.18`でした。Yosemiteには`FTDIUSBSerialDriver_10_4_10_5_10_6_10_7`をインストールします。

### Gort

[Gort](http://gort.io/)はArduinoやSparkCoreなどのコネクテッドデバイスのファームウェアを更新するCLIです。OSXの64bit版をダウンロードしてインストールします。Goで書かれているのでバイナリをコピーするだけです。

``` bash
$ cd ~/Downloads
$ wget https://s3.amazonaws.com/gort-io/0.3.0/gort_0.3.0_darwin_amd64.zip
$ unzip gort_0.3.0_darwin_amd64.zip
Archive:  /Users/mshimizu/Downloads/gort_0.3.0_darwin_amd64.zip
  inflating: gort_0.3.0_darwin_amd64/gort
  inflating: gort_0.3.0_darwin_amd64/README.md
  inflating: gort_0.3.0_darwin_amd64/LICENSE
$ sudo cp gort_0.3.0_darwin_amd64/gort /usr/local/bin/
```

バージョンを確認します。

``` bash
$ gort --version
gort version 0.3.0
```

### Firmata

[Firmata](http://arduino.cc/en/reference/firmata)ファームウェアをインストールします。Firmataはホストマシンからシリアル通信を使ってArduinoを制御するためのプロトコルです。まずArduino UnoをUSBケーブルで接続します。

``` bash
$ gort scan serial
/dev/cu.Bluetooth-Incoming-Port		/dev/tty.Bluetooth-Incoming-Port
/dev/cu.Bluetooth-Modem			/dev/tty.Bluetooth-Modem
/dev/cu.iPhoneDragonfly-Wireles		/dev/tty.iPhoneDragonfly-Wireles
/dev/cu.usbmodem1421			/dev/tty.usbmodem1421
```

Arduinoはavrdudeを使ってファームウェアを書き込むのでホストマシンにavrdudeをインストールします。

``` bash
$ gort arduino install
Attempting to install avrdude with Homebrew.
```

ArduinoにFirmataファームウェアをアップロードします。

``` bash
$ gort arduino upload firmata /dev/tty.usbmodem1421
avrdude: AVR device initialized and ready to accept instructions
...
avrdude: verifying ...
avrdude: 11452 bytes of flash verified

avrdude done.  Thank you.
```

### Node.js

OSXの場合[nodebrew](https://github.com/hokaccha/nodebrew)を使ってNode.jsをインストールするのが多いようですが、Ubuntuと同じように[nvm](https://github.com/creationix/nvm)を使います。インストールスクリプトをダウンロードして実行します。

``` bash
$ curl https://raw.githubusercontent.com/creationix/nvm/v0.24.0/install.sh | bash
```

自動的に`.bash_profile`にnvm起動スクリプトの実行が追加されました。

``` bash ~/.bash_profils
export NVM_DIR="/Users/mshimizu/.nvm"
[ -s "$NVM_DIR/nvm.sh" ] && . "$NVM_DIR/nvm.sh"  # This loads nvm
```

シェルを開き直し、nvmのバージョンを確認します。

``` bash
$ nvm --version
0.24.0
```

Node.jsは0.10の最新版をインストールします。

``` bash
$ nvm install 0.10
######################################################################## 100.0%
Now using node v0.10.37
```

インストールしたNode.jsを有効にします。npmのバージョンも確認します。

``` bash
$ nvm use 0.10
Now using node v0.10.37
$ npm -v
1.4.28
```

nvmを有効にして直接nodeのコマンドを実行することもできます。

``` bash
$ nvm run 0.10 --version
Running node v0.10.37
v0.10.37
```

## cylon-firmata

ホストマシンのOSXにCylon.jsの[Firmataアダプター](https://github.com/hybridgroup/cylon-firmata)をインストールします。最初にプロジェクトを作成します。

``` bash
$ mkdir -p ~/node_apps/firmata-led
$ cd !$
```

package.jsonに必要なパッケージを定義します。

``` json ~/node_apps/firmata-led/package.json
{
  "name": "firmata-led",
  "version": "0.0.1",
  "private": true,
  "dependencies": {
    "cylon-firmata": "0.19.0"
  },
  "scripts": {"start": "node app.js"}
}
```

`npm install`でパッケージをインストールします。

``` bash
$ npm install
```

### Lチカ

Lチカするプログラムを用意します。`port`は`gort scan serial`で確認した`/dev/tty.usbmodem1421`を指定します。

``` js ~/node_apps/firmata-led/app.js
var Cylon = require('cylon');

Cylon.robot({
  connections: {
    arduino: { adaptor: 'firmata', port: '/dev/tty.usbmodem1421' }
  },

  devices: {
    led: { driver: 'led', pin: 13 }
  },

  work: function(my) {
    every((1).second(), my.led.toggle);
  }
}).start();
```

`npm start`でプログラムを実行します。Arduino Unoの13番ピンのLEDがLチカします。

``` bash
$ npm start

> firmata-led@0.0.1 start /Users/mshimizu/node_apps/firmata-led
> node app.js

I, [2015-03-15T03:19:30.103Z]  INFO -- : [Robot 45547] - Initializing connections.
I, [2015-03-15T03:19:30.248Z]  INFO -- : [Robot 45547] - Initializing devices.
I, [2015-03-15T03:19:30.249Z]  INFO -- : [Robot 45547] - Starting connections.
I, [2015-03-15T03:19:33.513Z]  INFO -- : [Robot 45547] - Starting devices.
I, [2015-03-15T03:19:33.514Z]  INFO -- : [Robot 45547] - Working.
```

