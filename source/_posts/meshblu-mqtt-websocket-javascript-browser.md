title: "MeshbluのRESTとMQTTのメッセージをWebブラウザで直接受信する"
date: 2015-04-26 15:00:36
categories:
 - IoT
tags:
 - JavaScript
 - Meshblu
 - MQTT
 - WebSocket
description: MeshbluにはWebブラウザで使うmeshblu.jsのJavaScriptライブラリがあります。ArduinoなどからMQTTでpublishしたセンシングデータを通常のWebブラウザ上で受信できるようになります。JavaScriptが使える環境は多いのでコネクテッドデバイス間のメッセージングの選択肢が広がります。
---

MeshbluにはWebブラウザで使う[Meshblu.js](http://meshblu.octoblu.com/js/meshblu.js)のJavaScriptライブラリがあります。ArduinoなどからMQTTでpublishしたセンシングデータを通常のWebブラウザ上で受信できるようになります。JavaScriptが使える環境は多いのでコネクテッドデバイス間のメッセージングの選択肢が広がります。


<!-- more -->

## デバイスの作成

Meshbluのデバイス情報の`protocol`キーに使用しているプロトコルが記述されています。同じデバイスを使ってWebSocketとMQTTのメッセージ送信をする場合はprotocolを書き換える必要があります。WebSocketとMQTTのテスト用にデバイスを２つ用意します。

WebSocket用のデバイスを作成します。

``` bash
$ curl -X POST \
  "http://localhost/devices" \
  -d "name=websocket-pubsub&uuid=websocket-pubsub&protocol=websocket&token=A9plQrTH"
```

MQTT用のデバイスを作成します。

``` bash
$ curl -X POST \
  "http://localhost/devices" \
  -d "name=mqtt-pub&uuid=mqtt-pub&protocol=mqtt&token=RdHBvvYZ"
```

## index.html

Meshbluのブローカーからメッセージを受信するJavaScriptを書きます。ローカルの適当な場所に配置するだけです。WebSocketでSubscribeするデバイスは`websocket-pubsub`をIDに持ちます。createConnectionの`server`はMeshbluが起動しているホスト名またはIPアドレスを指定します。

```html ~/Documents/meshblu/index.html
<!DOCTYPE html>
<html lang="ja">
  <head>
    <meta charset="utf-8">
    <title>Meshblu.js</title>
    <script src="http://meshblu.octoblu.com/js/meshblu.js"></script>
    <script src="http://ajax.googleapis.com/ajax/libs/jquery/2.1.3/jquery.min.js"></script>
    <script>
      var conn = meshblu.createConnection({
        "uuid": "websocket-pubsub",
        "token": "A9plQrTH",
        "server": "{MESHBLU_BROKER}}",
        "port": 80
      });
      conn.on('ready', function(data){
        console.log('UUID AUTHENTICATED!');
        console.log(data);
        conn.on('message', function (message) {
          console.log('message received', message);
          $(".activity").prepend(JSON.stringify(message) + '<br />');
        });
      });
    </script>
  </head>
  <body>
   <p>Subscribe Messages</p>
   <div class="activity"></div>
  </body>
</html>
```

## WebSocketとMQTTのテスト

日本語を含んだpayloadがWebブラウザで正常に表示されるかも試します。Chromeブラウザから先ほど作成したローカルのindex.htmlファイルを開きます。

最初にcurlコマンドからメッセージをHTTP POSTします。payloadに"川崎"という日本語を追加しました。

```
$ curl -X POST  \
  http://localhost/messages  \
  -d '{"devices": ["websocket-pubsub"], "payload": {"temperature":25,"city":"川崎"}}'  \
  --header "meshblu_auth_uuid: websocket-pubsub"  \
  --header "meshblu_auth_token: A9plQrTH"
```

次にMosquittoからメッセージをMQTT publishします。同様にpayloadには日本語を含んでいます。

``` bash
$ mosquitto_pub \
  -h localhost  \
  -p 1883 \
  -t message \
  -m '{"devices": "websocket-pubsub", "payload": {"temperature":26,"city":"横浜"}}' \
  -u mqtt-pub \
  -P RdHBvvYZ \
  -d
```

ブラウザに2つの日本語を含むメッセージが正常に表示されました。

![Meshblujs.png](/2015/04/26/meshblu-mqtt-websocket-javascript-browser/Meshblujs.png)



