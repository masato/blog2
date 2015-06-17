title: "Arduinoから大気圧センサーモジュールBMP180のデータをMeshbluにPOSTする"
date: 2015-03-21 12:42:34
tags:
 - Meshblu
 - BMP180
 - Arduino
 - Cylonjs
 - NeDB
 - Firmata
 - センサー
description: Docker上に構築したMeshbluへArduinoからセンシングデータを送信してみます。前回起動したコンテナではMongoDBを使わない設定なので、POSTしたデータはNeDBにJSONファイルとして保存されます。大気圧センサーモジュールはBMP180を使います。前回Intel Edisonでこのモジュールを使ったときはcylon-i2cやmraaが使えず、bmp085パッケージを使いました。Arduinoの場合cylon-firmataが動作するようです。Cylon.jsを使ってセンシングデータを取得してMeshbluにデータをPOSTしてみます。
---

[Docker上に構築](/2015/03/17/meshblu-on-docker-install/)したMeshbluへArduinoからセンシングデータを送信してみます。前回起動したコンテナではMongoDBを使わない設定なので、POSTしたデータは[NeDB](https://github.com/louischatriot/nedb/)にJSONファイルとして保存されます。大気圧センサーモジュールは[BMP180](https://www.switch-science.com/catalog/1598/)を使います。[前回](/2015/03/14/intel-edison-ssci-eaglet-mft-bmp180-i2c/)Intel Edisonでこのモジュールを使ったときは[cylon-i2c](https://github.com/hybridgroup/cylon-i2c)や[mraa](https://github.com/intel-iot-devkit/mraa)が使えず、[bmp085](https://github.com/fiskeben/bmp085)パッケージを使いました。Arduinoの場合[cylon-firmata](https://github.com/hybridgroup/cylon-firmata)が動作するようです。[Cylon.js](http://cylonjs.com/)を使ってセンシングデータを取得してMeshbluにデータをPOSTしてみます。

<!-- more -->

## cylon-skynetが古い

Cylon.jsからMeshbluのAPIを使う場合、[Skynet](http://cylonjs.com/documentation/platforms/skynet/)モジュールを使います。名前が古いようにdependenciesのパッケージも古くなっています。


### clone

とりあえずリポジトリをcloneして[masato/cylon-skynet](https://github.com/masato/cylon-skynet)しました。`lib/adaptor.js`と`package.json`を修正します。

``` bash
$ git diff HEAD^^
diff --git a/lib/adaptor.js b/lib/adaptor.js
index 4f8a361..18aecb9 100644
--- a/lib/adaptor.js
+++ b/lib/adaptor.js
@@ -8,7 +8,8 @@

 "use strict";

-var Skynet = require("skynet"),
+//var Skynet = require("skynet"),
+var Skynet = require("meshblu"),
     Cylon = require("cylon");

 var Adaptor = module.exports = function Adaptor(opts) {
@@ -48,7 +49,8 @@ Adaptor.prototype.connect = function(callback) {
   this.connector = Skynet.createConnection({
     uuid: this.uuid,
     token: this.token,
-    host: this.host,
+    //host: this.host,
+    server: this.host,
     port: this.portNumber,
     forceNew: this.forceNew
   });
@@ -102,3 +104,7 @@ Adaptor.prototype.message = function(data) {
 Adaptor.prototype.subscribe = function(data) {
   return this.connector.subscribe(data);
 };
+
+Adaptor.prototype.data = function(data,fn) {
+    return this.connector.data(data,fn);
+};
diff --git a/package.json b/package.json
index d076c05..6c0a386 100644
--- a/package.json
+++ b/package.json
@@ -29,6 +29,6 @@

   "dependencies": {
     "cylon":  "~0.22.0",
-    "skynet": "latest"
+    "meshblu": "latest"
   }
 }
```

### Node.jsプログラムのデバイス/サービス登録

MeshbluではIoTプラットフォームにつながる、人、モノ、サービスをすべてUUIDとTOKENで管理します。そのため操作する人も、コネクテッドデバイスも、連携するWebサービスも対等にUUIDとTOKENを発行する必要があります。今回はOSXのNode.jsプログラムをサービスとして登録します。`MESHBLU_URL`はTLS/SSL通信するMeshbluサーバーです。オレオレ証明書を使っているので`--insecure`フラグが必要です。

``` bash
$ export MESHBLU_URL=https://{Meshbluのドメイン名}:4443
$ curl -X POST \
  -d "name=iot-http" \
  --insecure \
  "$MESHBLU_URL/devices"
{"uuid":"xxx","online":false,"timestamp":"2015-03-23T03:46:19.254Z","name":"mqtt-osx","type":"host","ipAddress":"172.17.42.1","token":"xxx"}
```

## BMP180のサンプル

プロジェクトを作成します。

``` bash
$ mkdir -p ~/node_apps/firmata-bmp180-meshblu/
$ cd !$
```

package.jsonの`cylon-skynet`はcloneして修正した[リポジトリ](https://github.com/masato/cylon-skynet/)を指定します。

```json ~/node_apps/firmata-bmp180-meshblu/package.json
{
  "name": "firmata-bmp180-meshblu",
  "version": "0.0.1",
  "private": true,
  "dependencies": {
    "cylon-firmata": "0.19.0",
    "cylon-skynet": "masato/cylon-skynet"
  },
  "scripts": {"start": "node app.js"}
}
```

### app.js 

Arduinoのデバイスファイルは、MacBook Proの右側のUSBポートにを挿したので`/dev/tty.usbmodem1421`になりました。`host`、`uuid`、`token`はそれぞれ上記のデバイス登録で取得した値を使います。

``` js ~/node_apps/firmata-bmp180-meshblu/app.js 
"use strict";
var Cylon = require("cylon");
var host = "wss://{Meshbluのドメイン名}";
var portNumber = 4443;
var uuid = "{Node.jsプログラムのUUID}}";
var token = "{Node.jsプログラムのTOKEN}}";
Cylon.robot({
    connections: {
        arduino: { adaptor: "firmata", port: "/dev/tty.usbmodem1421" },
        skynet: { adaptor: 'skynet',
                  host: host,
                  portNumber: portNumber,
                  uuid: uuid,
                  token: token}
    },
    devices: {
        yellow: { driver: 'led', pin: 13, connection: 'arduino' },
        bmp180: { driver: "bmp180" }
    },
    work: function(my) {
        my.skynet.on("message", function(data) {
            console.log('data: ' + JSON.stringify(data));
            if(data.payload.yellow === "on") {
                my.yellow.turnOn();
            } else if(data.payload.yellow === "off") {
                my.yellow.turnOff();
            }
        });

        every((5).seconds(), function() {
            my.bmp180.getAltitude(1, null, function(err, val) {
                if (err) {
                    console.log(err);
                    return;
                 }
                 my.skynet.data({
                    "temperature": val.temp,
                    "pressure": val.press,
                    "altitude": val.alt
                 },function(data){
                    console.log(data);
		             });
            });
	      });
    }
}).start();
```

`connections`ディレクティブにはArduinoとMeshblu(Skynet)の2つを定義します。`devices`にはMeshbluからメッセージをsubscribeしてLチカするLEDと、大気圧センサーモジュールのBMP180を定義します。Arduinoが`on`のメッセージを受信するとLEDが点灯し、`off`で消灯します。BMP180からは5秒間隔でセンシングします。温度(temperature)、大気圧(pressure)、標高(altitude)のデータをJSON形式にフォーマットして、Meshbluに送ります。


## サンプルの起動

Node.jsのサンプルプログラムを起動します。`my.skynet.data()`のコールバックで実装している標準出力が表示されます。

``` bash
$ nvm use 0.10
Now using node v0.10.37
$ npm start
...
{ timestamp: '2015-03-23T06:33:15.610Z',
  temperature: '28.6',
  pressure: '100153',
  altitude: '98.03716036353367' }
{ timestamp: '2015-03-23T06:33:20.615Z',
  temperature: '28.6',
  pressure: '100146',
  altitude: '98.62549086806833' }
{ timestamp: '2015-03-23T06:33:25.619Z',
  temperature: '28.6',
  pressure: '100150',
  altitude: '98.2892979312604' }
```


### Meshblu側のデータベース

今回のMeshbluの構成ではMongoDBの利用をコメントアウトしているので、センシングデータはNode.jsの組み込みデータベースの[NeDB](https://github.com/louischatriot/nedb/)にJSONファイルとして保存されました。

Meshbluコンテナを動かしているDockerホストにログインして、ボリュームにアタッチしている作業ディレクトリに移動します。

```
$ cd ~docker_apps/meshblu-dev
```

センシングデータは`data.db`ファイルに生のJSONとして保存されています。

``` bash
$ tail data.db
{"timestamp":"2015-03-23T06:33:15.610Z","temperature":"28.6","pressure":"100153","altitude":"98.03716036353367","_id":"krDon5OH6Frs17ic"}
{"timestamp":"2015-03-23T06:33:20.615Z","temperature":"28.6","pressure":"100146","altitude":"98.62549086806833","_id":"JGdcd8c4ycXXlKEk"}
{"timestamp":"2015-03-23T06:33:25.619Z","temperature":"28.6","pressure":"100150","altitude":"98.2892979312604","_id":"lMj4o7MaYB3RmLrW"}
```

### Pub/Subのテスト

次はメッセージのPub/Subのテストをします。テストに使う環境変数を作成します。

``` bash
$ export MESHBLU_URL=https://{Meshbluのドメイン名}:4443
$ export PUB_DEV_UUID={Node.jsプログラムのUUID}
$ export PUB_DEV_TOKEN={Node.jsプログラムのTOKEN}
```

`--header`には認証情報としてNode.jsプログラムのUUID/TOKENを指定します。送信データの`devices`キーに、メッセージ送信先デバイスのUUIDを指定します。今回は認証UUIDと送信先UUIDが同じなので自分自身にメッセージを送るサンプルになります。`payload`キーにはデバイスに送信するメッセージの本文をJSON形式で指定します。`on`をデバイスが受信するとLEDを点灯(`my.yellow.turnOn();`)します。

``` bash
$ curl --insecure -X POST  \
  "$MESHBLU_URL/messages" \
   -d '{"devices": ["'"$PUB_DEV_UUID"'"], "payload": {"yellow":"on"}}'  \
   --header "meshblu_auth_uuid: $PUB_DEV_UUID"  \
   --header "meshblu_auth_token: $PUB_DEV_TOKEN"
{"devices":["xxx"],"payload":{"yellow":"on"}}
```

`off`のメッセージを受信するとLEDを消灯(`my.yellow.turnOff();`)します。

``` bash
$ curl --insecure -X POST  \
  "$MESHBLU_URL/messages"  \
   -d '{"devices": ["'"$PUB_DEV_UUID"'"], "payload": {"yellow":"off"}}'  \
   --header "meshblu_auth_uuid: $PUB_DEV_UUID"  \
   --header "meshblu_auth_token: $PUB_DEV_TOKEN"
{"devices":["xxx"],"payload":{"yellow":"off"}}
```
