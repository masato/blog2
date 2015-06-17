title: "Node-RED on Docker - Part2: MQTTブローカーのMoscaを使う"
date: 2015-02-08 22:04:52
tags:
 - Node-RED
 - Nodejs
 - Mosca
 - MQTT
description: 前回構築したNode-REDのinputにMQTTブローカーにMQTT.jsで実装されたMoscaを使ってみます。スタンドアロンとExpressなどのアプリに組み込む形で動作します。Node-REDのコンテナと同様にMoscaもDockerコンテナを用意します。
---


[前回構築した](/2015/02/07/node-red-on-docker-install/)Node-REDのinputにMQTTブローカーにMQTT.jsで実装されたMoscaを使ってみます。スタンドアロンとExpressなどのアプリに組み込む形で動作します。Node-REDのコンテナと同様にMoscaもDockerコンテナを用意します。

<!-- more -->

## Moscaのコンテナ

MQTTブローカーは[matteocollina/mosca](https://registry.hub.docker.com/u/matteocollina/mosca/)イメージからコンテナを起動します。ポートはDockerホストにマップします。

``` bash
$ docker pull matteocollina/mosca
$ docker run --rm --name mosca \
  -p 80:80 \
  -p 1883:1883 \
  matteocollina/mosca
{"name":"mosca","hostname":"73bdd75f5d2e","pid":1,"level":30,"mqtt":1883,"http":80,"msg":"server started","time":"2015-02-08T00:19:09.719Z","v":0}
```

## Node-REDの設定

[前回構築した](/2015/02/07/node-red-on-docker-install/)Node-REDのinputにMoscaのMQTTブローカーを設定します。左ペインのinputからmqttを選択し、中央ペインにドラッグ&ドロップします。

* Broker: 10.1.3.67:1883
* username: admin
* password: password
* topic: message

![node-red-mqtt-in.png](/2015/02/08/node-red-on-docker-mqtt-mosca/node-red-mqtt-in.png)

左ペインのoutputからdebugをドラッグ&ドロップしてmqttとdebugを線で結びます。

![node-red-mqtt-out.png](/2015/02/08/node-red-on-docker-mqtt-mosca/node-red-mqtt-out.png)

右上のデプロイボタンを押します。MQTTブローカーと接続に成功したので、mqttのnodeの下にconnectedと表示されます。


## MQTTの通信テスト

### publisherコンテナ

テスト用にNode.js用のコンテナを作ります。[google/nodejs-runtime](https://registry.hub.docker.com/u/google/nodejs-runtime/)からDockerイメージをビルドするためのプロジェクトを作成します。

```bash
$ mkdir -p ~/docker_apps/publisher
```

package.jsonを用意します。

``` js package.json
{
  "name": "mqtt-publisher",
  "description": "mqtt publisher test app",
  "version": "0.0.1",
  "private": true,
  "dependencies": {
    "mqtt": "*"
  },
  "scripts": {"start": "node app.js"}
}
```

エントリーポイントのスクリプトを書きます。URLの書式は以下です。Node-REDで作成した`message`topicにpublishします。

* URL: mqtt://{username}:{password}@{broker}:{broker_port}

``` js app.js
var mqtt = require('mqtt')
  , client = mqtt.connect('mqtt://admin:password@10.1.3.67:1883');

setInterval(function() {
  client.publish('message');
  client.publish('message', 'foo');
  client.publish('message', Date.now().toString());
}, 1000);
```

Dockerfileを作成して、`docker build`を実行します。

``` bash
$ echo FROM google/nodejs-runtime > Dockerfile
$ docker build -t publisher .
```

### publisherコンテナの起動

使い捨てのコンテナを起動して、1880ポートをDockerホストにマップします。

``` bash
$ docker run --rm --name publisher publisher
> mqtt-publisher@0.0.1 start /app
> node app.js
```

ブラウザで動作確認をします。outputのdebug nodeにブローカーが受信したメッセージが表示されます。

![node-red-mqtt-test.png](/2015/02/08/node-red-on-docker-mqtt-mosca/node-red-mqtt-test.png)

Moscaコンテナの標準出力にも受信したメッセージが表示されます。

```
{"name":"mosca","hostname":"dd8f320575e9","pid":1,"client":"mqtt_501a7060.afe59","level":30,"topic":"message","msg":"unsubscribed","time":"2015-02-08T12:53:54.576Z","v":0}
{"name":"mosca","hostname":"dd8f320575e9","pid":1,"client":"mqtt_501a7060.afe59","level":30,"msg":"closed","time":"2015-02-08T12:53:54.576Z","v":0}
```


