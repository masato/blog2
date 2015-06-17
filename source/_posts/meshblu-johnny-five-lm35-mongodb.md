title: "Johnny-Fiveを使いLM35の温度データをArduinoからMeshbluのMongoDBに保存する"
date: 2015-03-24 13:00:08
tags:
 - Arduino
 - Johnny-Five
 - Meshblu
 - Nodejs
 - WebSocket
 - LM35
 - センサー
 - MongoDB
description: 先日構築したMeshbluコンテナのデータストアはNeDBのままでした。センシングデータをMongoDBに保存するように変更します。引き続きArduinoはFirmataとJohnny-Fiveを使いホストマシンから操作します。LM35を搭載した温度センサーモジュールを用意してArduinoから温度データを取得します。取得したデータはWebSocket over SSLでMeshbluにPOSTしてMongoDBに保存します。
---

先日構築した[Meshbluコンテナ](/2015/03/17/meshblu-on-docker-install)のデータストアは[NeDB](https://github.com/louischatriot/nedb/)のままでした。センシングデータをMongoDBに保存するように変更します。引き続きArduinoはFirmataとJohnny-Fiveを使いホストマシンから操作します。LM35を搭載した[温度センサーモジュール](http://www.aitendo.com/product/10763)を用意してArduinoから温度データを取得します。取得したデータはWebSocket over SSLでMeshbluにPOSTしてMongoDBに保存します。

<!-- more -->

## MongoDBをデータストアに使う

### Dockerコンテナの起動

Meshbluの[リポジトリ](https://github.com/octoblu/meshblu)を[fork](https://github.com/masato/meshblu)してDockerfileなど修正しました。forkしたリポジトリをcloneします。

``` bash
$ git clone https://github.com/masato/meshblu
$ cd meshblu
$ docker build -t meshblu .
```

Dockerコンテナにカレントディレクトリをマウントして使うため`node_modules`のシムリンクを作成します。

``` bash
$ ln -s /dist/node_modules ./node_modules
```

今回のMongoDBはMeshbluのコンテナ内で起動しています。環境変数`MONGODB_URI`にURLを指定します。

``` bash
$ docker run -d --name meshblu \
  -p 3000:3000 \
  -p 4443:4443 \
  -p 5683:5683 \
  -p 1883:1883 \
  -e PORT=3000 \
  -e MQTT_PORT=1883 \
  -e MQTT_PASSWORD=skynetpass \
  -e MONGODB_URI=mongodb://localhost:27017/skynet \
  -e SSL_PORT=4443 \
  -e SSL_CERT=/opt/meshblu/certs/server.crt \
  -e SSL_KEY=/opt/meshblu/certs/server.key \
  -v $PWD:/var/www \
  -v $PWD/docker:/etc/supervisor/conf.d \
  -v $HOME/docker_apps/certs/meshblu:/opt/meshblu/certs \
  meshblu
```

Meshbluが起動しました。

``` bash
$ docker logs meshblu
...
Meshblu (formerly skynet.im) development environment loaded...
...
Starting HTTP/HTTPS... done.
Starting MQTT... done.
CoAP listening at coap://localhost:5683
HTTP listening at http://0.0.0.0:3000
HTTPS listening at https://0.0.0.0:4443
...
MQTT listening at mqtt://0.0.0.0:1883
```

### 任意のuuid/tokenでデバイスの登録

statusを取得してMeshbluが起動していることを確認します。

``` bash
$ export MESHBLU_URL=https://xxxx
$ curl --insecure "$MESHBLU_URL/status"
{"meshblu":"online"}
```

センシングデータデバイスを登録します。普通に登録すると以下のようにuuidとtokenが取得できますがとても長いので扱いずらいです。

``` bash
$ curl -X POST \
  --insecure \
  -d "name=osx" \
  "$MESHBLU_URL/devices"
{"geo":null,"ipAddress":"172.17.42.1","name":"osx","online":false,"timestamp":"2015-03-24T07:34:12.997Z","uuid":"2aa8fd10-d1f8-11e4-a109-7b68e78fda3b","token":"b4f6c6df56b82f747444b891e133ddd8c34992b6"}
```

POSTするメッセージに任意の`uuid`と`token`を指定することができます。

``` bash
$ curl -X POST \
  --insecure \
  -d "name=johnny-five-test&uuid=johnny&token=five" \
  "$MESHBLU_URL/devices"
{"geo":null,"ipAddress":"172.17.42.1","name":"johnny-five-test","online":false,"timestamp":"2015-03-25T04:50:17.949Z","uuid":"johnny","token":"five"}
```

今回はArduinoのホストマシンにOSXを使います。さきほど取得したuuidとtoken、MeshbluのURLを`~/.bash_profile`に記述して再読込します。

``` bash ~/.bash_profile
$ export MESHBLU_URL=https://xxx
$ export OSX_UUID=johnny
$ export OSX_TOKEN=five
```

環境変数を指定して登録したデバイスを表示してみます。

``` bash
$ source ~/.bash_profile
$ curl -X GET \
  "$MESHBLU_URL/devices" \
  --insecure  \
  --header "meshblu_auth_uuid: $OSX_UUID" \
  --header "meshblu_auth_token: $OSX_TOKEN"
{"devices":[{"geo":null,"ipAddress":"172.17.42.1","name":"johnny-five-test","online":false,"timestamp":"2015-03-25T04:50:17.949Z","uuid":"johnny"}]}
```

### MongoDBのコレクションの確認

Dockerコンテナに入りMongoDBのコレクションを表示してみます。

``` bash
$ docker exec -it  meshblu bash
```

データベース名は環境変数`MONGODB_URI`に指定した`skynet`です。

``` bash
$ mongo skynet
MongoDB shell version: 2.4.13
connecting to: skynet
> show collections;
devices
system.indexes
```

先ほど登録したデバイスのコレクションが作成されています。
 
``` bash
> db.devices.find();
{ "_id" : ObjectId("55123e89a90c660a00775d53"), "geo" : null, "ipAddress" : "172.17.42.1", "name" : "johnny-five-test", "online" : false, "timestamp" : ISODate("2015-03-25T04:50:17.949Z"), "token" : "$2a$08$O3hl4gnoTHG0zj/rcIACieEqzB5zHsnktZwNDdFldrJqdVcBidVe6", "uuid" : "johnny" }
```

## センシングデータをPOSTする

OSXをホストマシンとしてFirmataプロトコルでArduinoに接続します。[Johnny-Five](https://github.com/rwaldron/johnny-five)からArduinoを操作してセンシングデータを取得します。取得したデータは[meshblu-npm](https://github.com/octoblu/meshblu-npm)を使ってMeshbluにPOSTしてMongoDBに保存します。

### LM35 温度センサモジュール

温度センサモジュールはaitendoから[M35DZ-3P](http://www.aitendo.com/product/10763)を購入しました。Jonny-Fiveから操作できる[LM35](https://github.com/rwaldron/johnny-five/blob/master/docs/temperature-lm35.md)を搭載しています。回路図を参考にしてArduinoと接続します。

![temperature-lm35.png](/2015/03/24/meshblu-johnny-five-lm35-mongodb/temperature-lm35.png)

### サンプルプロジェクト

OSX上にサンプルプロジェクトを作成します。

``` bash
$ mkdir -p ~/node_apps/johnny-five-meshblu-lm35
$ cd !$
```

`package.json`にインストールするパッケージを記述します。

```json ~/node_apps/johnny-five-meshblu-lm35/package.json
{
    "name": "johnny-five-meshblu-lm35",
    "version": "0.0.1",
    "private": true,
    "dependencies": {
        "johnny-five": "0.8.49",
        "meshblu": "1.19.0"
    },
    "scripts": {"start": "node app.js"}
}
```

app.jsにメインプログラムを書きます。Meshbluの接続情報は`~/.bash_profile`に記述した環境変数を使います。Meshbluに接続したあとJohnny-FiveからLM35の温度データを取得します。`freq: 5000`として5秒間隔にします。温度データが取得できたらJSONにフォーマットしてMesubluにPOSTします。

```js ~/node_apps/johnny-five-meshblu-lm35/app.js
var five = require("johnny-five"),
    meshblu = require('meshblu');

var conn = meshblu.createConnection({
    "uuid": process.env.OSX_UUID,
    "token": process.env.OSX_TOKEN,
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

    five.Board().on("ready", function() {
        console.log("Ready...");
        var temperature = new five.Temperature({
            controller: "LM35",
            pin: "A0",
            freq: 5000
        });

        temperature.on("data", function(err, data) {
            console.log(data.celsius);
            conn.data({
                "temperature":data.celsius
            });
        });
    });
});
```

必要なパッケージをインストールしてNode.jsプログラムを起動します。標準出力には摂氏の温度が5秒間隔で表示されます。

``` bash
$ npm install
$ npm start
...
25.390625
25.390625
25.87890625
25.87890625
25.390625
```

### MongoDBのデータ確認

Meshbluコンテナを起動しているDockerホストにログインします。Meshbluコンテナに入りMongoDBの`data`コレクションをリストします。Arduinoに接続したLM35モジュールから取得した温度データが、OSXで動作しているJohnny-Fiveを通してクラウド上のMongoDBに保存されました。

``` bash
$ docker exec -it meshblu bash
mongo skynet
MongoDB shell version: 2.4.13
connecting to: skynet
> db.data.find();
{ "timestamp" : "2015-03-25T03:53:08.179Z", "temperature" : "25.390625", "_id" : ObjectId("55123124c2eede05005958f2") }
{ "timestamp" : "2015-03-25T03:53:13.176Z", "temperature" : "25.87890625", "_id" : ObjectId("55123129c2eede05005958f3") }
{ "timestamp" : "2015-03-25T03:53:18.251Z", "temperature" : "25.87890625", "_id" : ObjectId("5512312ec2eede05005958f4") }
```
