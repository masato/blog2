title: "Arduino UnoをUbuntuホストマシンからCylon.jsとFirmataを使って操作する"
date: 2015-02-23 22:36:51
tags:
 - ArduinoUno
 - Cylonjs
 - Firmata
 - MQTT
 - Gort
description: FirmataはホストマシンからUSBシリアル通信を経由してArduinoを操作できるプロトコルです。Cylon.jsのcylon-firmataアダプタを使い、Node.jsからArduino Uno R3を動かす環境を用意しようと思います。Cylon.jsのフレームワークで抽象化してくれるのでBeagleBone BlackにデプロイしたNode.jsのプログラムがPIN番号を変更するだけで動きます。
---

[Firmata](http://arduino.cc/en/reference/firmata)はホストマシンからUSBシリアル通信を経由してArduinoを操作できるプロトコルです。Cylon.jsの[cylon-firmata](https://github.com/hybridgroup/cylon-firmata)アダプタを使い、Node.jsから[Arduino Uno R3](http://arduino.cc/en/Main/arduinoBoardUno)を動かす環境を用意しようと思います。Cylon.jsのフレームワークが抽象化してくれるのでBeagleBone BlackにデプロイしたNode.jsのプログラムがPIN番号を変更するだけで動きます。

<!-- more -->

## Ubuntu 14.04のXfceホストマシン

BeagleBone Blackと違いArduino Uno R3にはNode.jsをインストールしていないのでFirmataの場合Node.jsはホストマシン側にあります。ホストマシンにはChromebookにcroutonでインストールしたUbuntu 14.04のXfceデスクトップ環境を使います。

### Arduino IDE

ホストマシンのUbuntuにArduino IDEをインストールします。apt-getでインストールするバージョンは1.0.5と古いのですが、最新版の1.6.0のdebパッケージはARMv7環境にインストールが失敗してしまいました。

``` bash
$ sudo apt-get update && sudo apt-get install arduino arduino-core  
```

ArduinoをUSBケーブルでChromebookと接続すると`ttyACM0`のデバイスファイルが作成されます。

``` bash
$ ls -al  /dev/ttyACM0 
crw-rw---- 1 root serial 166, 0 Feb 23 13:34 /dev/ttyACM0
```

ログインユーザーでデバイスファイルを操作できるように権限設定をします。

``` bash
$ sudo chmod a+rw /dev/ttyACM0
```

インストールした`arduino`コマンド実行してArduino IDEを起動します。

``` bash
$ arduino
```

![arduino-ide.png](/2015/02/23/cylonjs-arduino-uno-firmata-gort/arduino-ide.png)


Arduinoのデバイスをシリアル接続で利用するためにdialogグループに追加するダイアログがでます。

![dialout-group.png](/2015/02/23/cylonjs-arduino-uno-firmata-gort/dialout-group.png)

ユーザをdialout グループに登録します。

``` bash
$ sudo usermod -aG dialout mshimizu
```

ログインし直なおしてグループを確認します。

``` bash
$ groups
mshimizu dialout video sudo plugdev audio
```

## Firmata

FirmataファームウェアをホストマシンからArduinoにインストールする方法は、Gortを使う場合とArduino IDEからアップロードする場合と2種類あります。

### Gortを使う場合

[Gort](http://gort.io/)はGoで書かれたロボティクスのためのコマンドラインツールです。Arduino Firmataと互換性があるSparkCoreやDigisparkも操作できるようです。今回はavrdudeをインストールしてFirmataをArduinoにアップロードします。まずホストマシンにGortをインストールします。

``` bash
$ cd ~/Downloads
$ wget https://s3.amazonaws.com/gort-io/0.3.0/gort_0.3.0_linux_arm.tar.gz
$ tar zxvf gort_0.3.0_linux_arm.tar.gz 
gort_0.3.0_linux_arm/gort
gort_0.3.0_linux_arm/README.md
gort_0.3.0_linux_arm/LICENSE
$ sudo cp gort_0.3.0_linux_arm/gort /usr/local/bin
```

Gortのバージョンを確認します。

``` bash
$ which gort
/usr/local/bin/gort
$ gort -v 
gort version 0.3.0
```

シリアルポートをscanしてArduinoのデバイスファイルを確認します。

``` bash
$ gort scan serial

1 serial port(s) found.

1. [/dev/ttyACM0] - [usb-Arduino__www.arduino.cc__0043_55431313338351E0F111-if00]
```

avrdudeをインストールしてsketchesアップロードの準備をします。

``` bash
$ gort arduino install
Attempting to install avrdude with apt-get.
Reading package lists... Done
Building dependency tree 
Reading state information... Done
avrdude is already the newest version.
avrdude set to manually installed.
0 upgraded, 0 newly installed, 0 to remove and 56 not upgraded.
```

FirmataファームウェアをArduinoにアップロードします。

``` bash
$ gort arduino upload firmata /dev/ttyACM0
 
avrdude: AVR device initialized and ready to accept instructions

Reading | ################################################## | 100% 0.01s

avrdude: Device signature = 0x1e950f
avrdude: reading input file "/tmp/555901447"
avrdude: writing flash (11452 bytes):

Writing | ################################################## | 100% 1.95s

avrdude: 11452 bytes of flash written
avrdude: verifying flash memory against /tmp/555901447:
avrdude: load data flash data from input file /tmp/555901447:
avrdude: input file /tmp/555901447 contains 11452 bytes
avrdude: reading on-chip flash data:

Reading | ################################################## | 100% 1.55s

avrdude: verifying ...
avrdude: 11452 bytes of flash verified

avrdude done. Thank you.
```

### Arduino IDEを使う場合

Arduino IDEを使ってFirmataをアップロードする場合は通常のsketchesのアップロードと同じです。まずメニューからサンプルからStandardFirmataのソースファイルを開きます。

* File > Examples > Firmata > StandardFirmata

メニューからデバイスを選択します。

* Tools  > Board > Arduino Uno

シリアルポートを選択します。

* Tools > Serial Port > /dev/ttyACM0

アップロードまたは右矢印ボタンをクリックしてArduinoにアップロードします。

* File > Upload

![standardfirmata-upload.png](/2015/02/23/cylonjs-arduino-uno-firmata-gort/standardfirmata-upload.png)


## Node.js

ホストマシンからFirmataを通してしてArduinoを操作するためにUbuntuにNode.jsをインストールします。いつもはnvmを使いますがChromebookがARMv7なのでソースからビルドする必要があります。

``` bash
$ nvm install -s v0.10
```

非力なChromebookなので今回は簡単にパッケージからインストールします。

``` bash
$ sudo apt-get install nodejs npm
$ sudo update-alternatives --install /usr/bin/node node /usr/bin/nodejs 10
```

Node.jsのバージョン確認します。比較的新しいNode.jsがインストールされました。

``` bash
$ node -v
v0.10.25
$ npm -v
1.3.10
```

## cylon-firmata

ホストマシンのUbuntuにCylon.jsのArduino Firmataアダプタである、[cylon-firmata](https://github.com/hybridgroup/cylon-firmata)をインストールします。package.jsonに必要なパッケージを定義します。MQTTでサンプルに使います。

``` json ~/node_apps/package.json
{
  "name": "mqtt-led",
  "version": "0.0.1",
  "private": true,
  "dependencies": {
    "cylon-firmata": "0.19.0",
    "cylon-mqtt": "0.4.0"
  },
  "scripts": {"start": "node app.js"}
}
```

`npm install`します。

``` bash
$ cd ~/node_apps/
$ npm install
```

### Lチカ

Lチカするサンプルアプリを用意しました。

``` js ~/node_apps/app.js
var Cylon = require('cylon');

Cylon.robot({
  connections: {
    arduino: { adaptor: 'firmata', port: '/dev/ttyACM0' }
  },

  devices: {
    led: { driver: 'led', pin: 13 }
  },

  work: function(my) {
    every((1).second(), my.led.toggle);
  }
}).start();
```

`npm start`でプログラムを実行するとLチカが始まります。

``` bash
$ npm start

> mqtt-led@0.0.1 start /home/mshimizu/node_apps
> node app.js

I, [2015-02-23T06:39:20.913Z] INFO -- : Initializing connections.
I, [2015-02-23T06:39:21.377Z] INFO -- : Initializing devices.
I, [2015-02-23T06:39:21.384Z] INFO -- : Starting connections.
I, [2015-02-23T06:39:26.465Z] INFO -- : Starting devices.
I, [2015-02-23T06:39:26.466Z] INFO -- : Working.
```


## MQTTとLチカ

### MQTTのpub/sub

workの中でMQTTのpub/subをすることでメッセージを送受信します。メッセージを受信するとLチカします。

``` js ~/node_apps/app.js
var Cylon = require('cylon');

// Initialize the robot
Cylon.robot({
  connections: {
    mqtt: { adaptor: 'mqtt', host: 'mqtt://xxx.xxx.xxx.xxx:1883' },
    arduino: { adaptor: 'firmata', port: '/dev/ttyACM0' }
  },

  devices: {
    toggle: { driver: 'mqtt', topic: 'toggle', connection: 'mqtt' },
    led: { driver: 'led', pin: '13', connection: 'arduino' }
  },

  work: function(my) {
    my.toggle.on('message',function(data) {
      console.log("Message on 'toggle': " + data);
      my.led.toggle();
    });

    every((1).second(), function() {
      console.log("Toggling LED.");
      my.toggle.publish('toggle');
    });
  }
}).start();
```

`npm start`でプログラムを実行します。ArduinoからMQTTのpub/subを通してLチカが始まりました。

``` bash
$ npm start
 npm start

> mqtt-led@0.0.1 start /home/mshimizu/node_apps
> node app.js

I, [2015-02-23T06:55:24.462Z]  INFO -- : Initializing connections.
I, [2015-02-23T06:55:25.031Z]  INFO -- : Initializing devices.
I, [2015-02-23T06:55:25.039Z]  INFO -- : Starting connections.
I, [2015-02-23T06:55:28.322Z]  INFO -- : Starting devices.
I, [2015-02-23T06:55:28.327Z]  INFO -- : Working.
Toggling LED.
Message on 'toggle': toggle
Toggling LED.
Message on 'toggle': toggle
Toggling LED.
Message on 'toggle': toggle
Toggling LED.
Message on 'toggle': toggle
Toggling LED.
Message on 'toggle': toggle
```

### IFTTTとMQTTとLチカ

BeagleBone BlackにデプロイしたIFTTTのDo Buttonを使ったサンプルを使ってみます。MQTTでメッセージを受信すると標準出力とLチカを3秒間行います。BeagleBone Blackのコードから変更したのは`connection`と`led`のPIN番号だけです。異なるコネクテッドデバイス間でもほぼ同じNode.jsのコードが実行できるのでCylon.jsは便利です。

``` js ~/node_apps/app.js
var Cylon = require('cylon');

// Initialize the robot
Cylon.robot({
  connections: {
    mqtt: { adaptor: 'mqtt', host: 'mqtt://xxx.xxx.xxx.xxx:1883' },
    arduino: { adaptor: 'firmata', port: '/dev/ttyACM0' }
  },

  devices: {
    ifttt: { driver: 'mqtt', topic: 'ifttt/bbb', connection: 'mqtt' },
    led: { driver: 'led', pin: '13', connection: 'arduino' }
  },

  work: function(my) {
    my.ifttt.on("message",function(data) {
      console.log("Message on 'ifttt': " + data);
      var timer = setInterval(my.led.toggle, 200);
      setTimeout(function(){ clearInterval(timer)},3000);
    });
  }
}).start();
```

`npm start`でプログラムを実行します。AndroidのDo ButtonアプリからWordPressボタンをタップするとMQTTブローカーを経由して、位置情報から生成したGoogleマップのURLがArduinoに通知されました。

``` bash
$ npm start

> mqtt-led@0.0.1 start /home/mshimizu/node_apps
> node app.js

I, [2015-02-23T06:51:37.312Z]  INFO -- : Initializing connections.
I, [2015-02-23T06:51:37.981Z]  INFO -- : Initializing devices.
I, [2015-02-23T06:51:37.998Z]  INFO -- : Starting connections.
I, [2015-02-23T06:51:41.334Z]  INFO -- : Starting devices.
I, [2015-02-23T06:51:41.338Z]  INFO -- : Working.
Message on 'ifttt': {"username":"username","password":"password","title":"","description":"<div><img src='http://maps.google.com/maps/api/staticmap?center=35.xxxxxxx,139.xxxxxxx&zoom=19&size=640x440&scale=1&maptype=roadmap&sensor=false&markers=color:red%7C35.xxxxxxx,139.xxxxxxx' style='max-width:600px;' /><br/><div>Do Button pressed on February 23, 2015 at 03:51PM http://ift.tt/1zBK1rg</div></div>","tags":{"string":"Do Button"},"post_status":"publish"}
```
