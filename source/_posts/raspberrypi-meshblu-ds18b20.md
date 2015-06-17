title: "Raspberry PiからMeshbluにDS18B20のセンシングデータを送信する"
date: 2015-04-02 15:47:29
tags:
 - RaspberryPi
 - Meshblu
 - DS18B20
 - Johnny-Five
 - Raspi-io
 - センサー
 - MongoDB
description: Firmata互換のIOを使い、同じセンサーモジュールからRaspberry PiでもArduinoでもセンシングデータ取得コードの抽象化を試しています。取得したデータはMeshbluに送信してMongoDBにストアします。特に1-WireのDS18B20の場合はArduinoでもAdd arduino library to Firmata or Johnny-Five 285によるとConfigurable Firmataが必要になります。GPIOのアナログとデジタルの違いもあってJohnny-Fiveを使った抽象化は難しそうです。
---

Firmata互換のIOを使い、同じセンサーモジュールからRaspberry PiでもArduinoでもセンシングデータ取得コードの抽象化を試しています。取得したデータはMeshbluに送信してMongoDBにストアします。特に1-WireのDS18B20の場合はArduinoでも[Add arduino library to Firmata or Johnny-Five #285](https://github.com/rwaldron/johnny-five/issues/285#issuecomment-31485445)によると[Configurable Firmata](https://github.com/firmata/arduino/tree/configurable)が必要になります。GPIOのアナログとデジタルの違いもあってJohnny-Fiveを使った抽象化は難しそうです。


<!-- more -->

## Raspi-ioの1-Wireは未実装

最初はJohnny-Fiveの[Raspi-io](https://github.com/bryan-m-hughes/raspi-io/)を試しました。[Temperature (DS18B20)](https://github.com/rwaldron/johnny-five/blob/master/docs/temperature-ds18b20.md)のサンプルのPIN番号をRaspberry Piにあわせて`GPIO4`に変更します。

``` js ~/node_apps/raspi-io-led/app.js
var five = require("johnny-five");

five.Board().on("ready", function() {
  // This requires OneWire support using the ConfigurableFirmata
  var temperature = new five.Temperature({
    controller: "DS18B20",
    //pin: 2
    pin: "GPIO4"
  });

  temperature.on("data", function(err, data) {
    console.log(data.celsius + "°C", data.fahrenheit + "°F");
  });
});
```

残念ながら実行するとエラーになりました。

``` bash
$ sudo npm start

> raspi-io-led@0.0.1 start /home/pi/node_apps/raspi-io-led
> node app.js

1428295981604 Device(s) undefined
1428295981745 Connected RaspberryPi-IO
1428295981825 Repl Initialized
>>
/home/pi/node_apps/raspi-io-led/node_modules/raspi-io/lib/index.js:720
        throw "sendOneWireConfig is not yet implemented";
        ^
sendOneWireConfig is not yet implemented
```

Raspi-ioの[index.js](https://github.com/bryan-m-hughes/raspi-io/blob/master/index.js)見ると1-Wireは未実装のようです。

```js index.js
  sendOneWireConfig() {
    throw 'sendOneWireConfig is not yet implemented';
  }
```

## Node.jsでw1_slaveファイルを読む

[前回](/2015/04/01/raspberrypi-1-wire-ds18b20/)はデバイスの`w1_slave`ファイルを読んでDS18B20から温度を読むことに成功しました。Node.jsでも直接`w1_slave`ファイルを読むことにします。[Publishing Temperature Reading From a DS18B20 1-Wire Digital Temperature Sensor Using a Raspberry Pi and node.js](http://jam.im/blog/2013/04/30/publishing-temperature-reading-from-a-ds18b20-1-wire-digital-temperature-sensor-using-a-raspberry-pi-and-node-dot-js/)を参考にします。

Rasbperry Piにログインしてプロジェクトを作成します。

``` bash
$ mkdir -p ~/node_apps/meshblu-ds18B20
$ cd !$
```

package.jsonに[meshblu-npm](https://github.com/octoblu/meshblu-npm)のパッケージを追加します。

```json ~/node_apps/meshblu-ds18B20/package.json
{
    "name": "meshblu-ds18b20",
    "version": "0.0.1",
    "private": true,
    "dependencies": {
        "meshblu": "1.19.1"
    },
    "scripts": {"start": "node app.js"}
}
```

環境変数に定義したMeshbluへの接続情報を確認します。Raspberry Piに用意してあった`mqtt-pub`デバイスを認証に使います。

``` bash ~/.bashrc
export MESHBLU_BROKER=xxx.xxx.xxx.x
export MESHBLU_URL=https://$MESHBLU_BROKER
export MQTT_PUB_UUID=mqtt-pub
export MQTT_PUB_TOKEN=123456
```

メインプログラムを書きます。DS18B20のデバイスIDはハードコードしました。Meshbluの認証が成功した後`w1_slave`ファイルを1分間隔で読みます。正規表現を使い温度の数字部分を抜き出します。摂氏にした温度はJSON形式にしてMeshbluへWebSocketで送信します。

``` js ~/node_apps/meshblu-ds18B20/app.js
var meshblu = require('meshblu'),
      fs = require('fs');

var interval = 60*1000;
var w1Slave = '/sys/bus/w1/devices/28-0314626a2cff/w1_slave';

var conn = meshblu.createConnection({
    "uuid": process.env.MQTT_PUB_UUID,
    "token": process.env.MQTT_PUB_TOKEN,
    "protocol": "websocket",
    "server": process.env.MESHBLU_URL
});

conn.on('notReady', function(data){
    console.log('UUID FAILED AUTHENTICATION!');
    console.log(data);
});

conn.on('ready', function(data){
    console.log('UUID AUTHENTICATED!');
    console.log(data);

    var readSensor = function(){    
        fs.readFile(w1Slave, 'utf8', function(err, data) {
            if (err) throw err;
            var matches = data.match(/t=(\d+)/);
            var celsius = parseInt(matches[1]) / 1000;
            var output = {"temperature": celsius};
            console.log(output);
            conn.data({
                "temperature": celsius
            });
        });
    };

    setInterval(readSensor, interval);    
});
```

プログラムを実行します。1間隔で取得した温度が標準出力されます。

``` bash
$ npm start

> meshblu-ds18b20@0.0.1 start /home/pi/node_apps/meshblu-ds18B20
> node app.js

UUID AUTHENTICATED!
{ api: 'connect',
  status: 201,
  socketid: 'Q2mR6Ctg5UOpJJrRAAAD',
  uuid: 'mqtt-pub',
  token: '889346' }
...
{ temperature: 27.375 }
{ temperature: 27.5 }
{ temperature: 27.625 }
{ temperature: 27.5 }
{ temperature: 27.625 }
```

## MeshbluコンテナのMongoDBの確認

Meshbluコンテナのシェルを起動します。

``` bash
$ docker exec -it  meshblu bash
```

MongoDBシェルを使いRaspberry Piから送信されたセンシングデータを確認します。

``` bash
$ mongo skynet
MongoDB shell version: 2.4.13
connecting to: skynet
```

直近の3件を表示します。1分間隔でセンシングデータが保存されています。

``` bash
> db.data.find().sort({ $natural: -1 }).limit(3)
{ "timestamp" : "2015-04-06T08:52:37.899Z", "temperature" : "27.625", "_id" : ObjectId("55224955a545260b00c971b3") }
{ "timestamp" : "2015-04-06T08:51:37.863Z", "temperature" : "27.5", "_id" : ObjectId("55224919a545260b00c971b2") }
{ "timestamp" : "2015-04-06T08:50:37.851Z", "temperature" : "27.625", "_id" : ObjectId("552248dda545260b00c971b1") }
```