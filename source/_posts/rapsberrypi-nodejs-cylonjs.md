title: "Rapsberry PiにNode.jsをインストールしてCylon.jsでLチカする"
date: 2015-02-27 20:24:52
tags:
 - RaspberryPi
 - Nodejs
 - Cylonjs
 - Lチカ
description: Raspberry Piにapt-getからNode.jsをインストールすると古いバージョンになります。ARMでNode.jsをソースからビルドするのは大変なので Error compiling v0.10.30 on Raspberry Pi Model Bなのでビルド済みのバイナリからインストールします。node-armのサイトではv0.10.36の最新バージョンまで公開されていますが、今回はNode.jsからarm-piのバイナリをダウンロードして使います。
---

Raspberry Piにapt-getからNode.jsをインストールすると古いバージョンになります。ARMでNode.jsをソースからビルドするのは大変([Error compiling v0.10.30 on Raspberry Pi Model B](https://github.com/joyent/node/issues/8062))なのでビルド済みのバイナリからインストールします。[node-arm](http://node-arm.herokuapp.com/)のサイトではv0.10.36の最新バージョンまで公開されていますが、今回は[Node.js](http://nodejs.org/dist)からarm-piのバイナリをダウンロードして使います。

<!-- more -->

## Node.jsのインストール

[Node.js](http://nodejs.org/dist/)に公開されているarm-piは[Why no more arm-pi binary release at http://nodejs.org/dist/ ?](https://github.com/joyent/node/issues/8098)にissueがあるように手動で公開しているようです。v0.10.28が最新でした。ビルド済みのバイナリをダウンロードして`/opt/node`にインストールします。

``` bash
$ wget http://nodejs.org/dist/v0.10.28/node-v0.10.28-linux-arm-pi.tar.gz
$ tar zxvf node-v0.10.28-linux-arm-pi.tar.gz
$ sudo mv node-v0.10.28-linux-arm-pi /opt/node
```

環境変数PATHにNode.jsを追加します。

``` bash
$ vi /home/pi/.bashrc
export PATH=/opt/node/bin:$PATH
$ source ~/.bashrc
```

バージョンを確認します。

``` bash
$ node -v
v0.10.28
$ npm -v
1.4.9
```

sudoでnpmコマンドが実行できるようにシムリンクを作成します。

``` bash
$ sudo ln -s /opt/node/bin/node /usr/bin/node
$ sudo ln -s /opt/node/bin/npm /usr/bin/npm
```

## Cylon.jsでLチカ

まずはプロジェクトを作成します。

``` bash
$ mkdir -p ~/node_apps/mqtt-led
$ cd !$
```

package.jsonに、Raspberry Piアダプタの[cylon-raspi](https://github.com/hybridgroup/cylon-raspi)を追加します。

``` json ~/node_apps/mqtt-led/package.json
{
  "name": "mqtt-led",
  "version": "0.0.1",
  "private": true,
  "dependencies": {
    "cylon-raspi": "0.15.0"
  },
  "scripts": {"start": "node app.js"}
}
```

`npm install`します。

``` bash
$ npm install
```

Cylon.jsの[Raspberry Pi](http://cylonjs.com/documentation/platforms/raspberry-pi/)のページにあるサンプルを実行します。

``` js ~/node_apps/mqtt-led/app.js
var Cylon = require("cylon");

Cylon.robot({
  connections: {
    raspi: { adaptor: 'raspi' }
  },

  devices: {
    led: { driver: 'led', pin: 11 }
  },

  work: function(my) {
    every((1).second(), my.led.toggle);
  }
}).start();
```

`npm start`をroot権限で実行するとLチカが始まります。

``` bash
$ sudo npm start
```
