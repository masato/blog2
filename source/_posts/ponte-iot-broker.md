title: "PonteのインストールとMQTT over SSL/TLS設定"
date: 2015-03-08 16:48:05
tags:
 - Ponte
 - Mosca
 - MQTT
 - CoAP
 - IoT
description: コネクテッドデバイス向けのMQTTブローカーにMoscaを使っていましたが、ブローカーにMQTT以外のプロトコルも使えると便利です。マルチプロトコルをサポートしていてMQTTにはMoscaを使っているPonteというブローカーがあるのでこちらを試してみます。PonteはEclipse FoundationのIoT Working Group、iot.eclipse.org配下のプロジェクトです。プロトコルにHTTP、MQTT、CoAPの3つをサポートしているのが特徴です。HTTPでpublishしてMQTTでsubscribeすることができます。
---

コネクテッドデバイス向けのMQTTブローカーに[Mosca](https://github.com/mcollina/mosca)を使っていましたが、ブローカーにMQTT以外のプロトコルも使えると便利です。マルチプロトコルをサポートしていてMQTTにはMoscaを使っているPonteというブローカーがあるのでこちらを試してみます。[Ponte](http://eclipse.org/ponte/)は[Eclipse Foundation](http://www.eclipse.org/)のIoT Working Group、[iot.eclipse.org](http://iot.eclipse.org/)配下のプロジェクトです。プロトコルにHTTP、MQTT、[CoAP](http://en.wikipedia.org/wiki/Constrained_Application_Protocol)の3つをサポートしているのが特徴です。HTTPでpublishしてMQTTでsubscribeすることができます。

<!-- more -->


## PonteのSSL/TLSサポート

SSL/TLSのサポートは直接されていませんが、MQTTはMoscaのconfigでプロトコルを指定することができます。HTTPは別途NginxでSSL Terminationをする予定です。今回はMQTT over SSL/TLSの設定を行います。

## Brokerコンテナ

BrokerコンテナのDockerイメージ用のプロジェクトを作成します。

``` bash
$ mkdir -p ~/docker_apps/ponte
$ cd !$
```

### Dockerfile

TLSで使う証明書はとりあえず暗号化できれば良いので、オレオレ証明書を作ります。またDockerベースイメージは[google/nodejs](https://registry.hub.docker.com/u/google/nodejs/)を使います。

``` bash ~/docker_apps/ponte/Dockerfile
FROM google/nodejs

RUN apt-get update && apt-get install -y libzmq-dev

WORKDIR /app
ONBUILD ADD package.json /app/
ONBUILD RUN npm install
ONBUILD ADD . /app

RUN npm install --unsafe-perm -g ponte bunyan
RUN mkdir -p /opt/nginx/certs \
 && cd /opt/nginx/certs \
 && openssl genrsa  -out server.key 4096 \
 && openssl req -new -batch -key server.key -out server.csr \
 && openssl x509 -req -days 3650 -in server.csr -signkey server.key -out server.crt

ADD config.js /app/
ENTRYPOINT ["ponte", "-c","/app/config.js","-v", "| bunyan"]
EXPOSE 8443 3000 5683
RUN apt-get clean && rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*
```

### npmのグローバルインストール

通常Node.jsのパッケージインストールはpackage.jsonに指定して`npm install`しますが、グローバルにインストールしようとするといろいろと問題があります。そのためpackage.jsonには指定せずDockerfileに記述します。

``` json ~/docker_apps/ponte/package.json
{
  "name": "ponte",
  "description": "ponte test app",
  "version": "0.0.1",
  "private": true
}
```

また`gyp WARN EACCES user "undefined" does not have permission`の警告がでてしまうため、`--unsafe-perm`フラグを付けます。

``` bash
RUN npm install --unsafe-perm -g ponte bunyan
```

### Ponteの設定ファイル

Ponte起動時にJSファイルで各プロトコルの設定を渡すことができます。MQTTにはオレオレ証明書と秘密鍵のパスを指定します。

``` js ~/docker_apps/ponte/config.js
module.exports = {
  logger: {
    level: "info",
  },
  persistence: {
    type: 'level',
    path: './db'
  },
  mqtt: {
    secure : {
      port: 8443,
      keyPath:"/opt/nginx/certs/server.key",
      certPath:"/opt/nginx/certs/server.crt"
    }
  }
}
```

### Ponteの起動

Dockerイメージのビルドとコンテナの起動をします。MQTT以外にもHTTPとCoAPのポートもDockerホストにマップしておきます。

``` bash
$ docker pull google/nodejs
$ docker build -t ponte .
$ docker run -d  --name ponte  \
  -p 8443:8443  \
  -p 3000:3000  \
  -p 5683:5683  \
  -it  ponte
```

ログを見ると3つのプロトコルで起動したことがわかります。

``` bash
$ docker logs -f ponte
{"name":"ponte","hostname":"17e232fd73f8","pid":1,"service":"MQTT","level":30,"mqtts":8443,"msg":"server started","time":"2015-03-08T04:57:26.910Z","v":0}
{"name":"ponte","hostname":"17e232fd73f8","pid":1,"service":"HTTP","level":30,"port":3000,"msg":"server started","time":"2015-03-08T04:57:26.918Z","v":0}
{"name":"ponte","hostname":"17e232fd73f8","pid":1,"service":"CoAP","level":30,"port":5683,"msg":"server started","time":"2015-03-08T04:57:26.919Z","v":0}
```

## Pub/Subクライアントのコンテナ

Pub/SubクライアントのDockerイメージを作成するためプロジェクトを作成します。

``` bash
$ mkdir -p ~/docker_apps/ponte-pubsub
$ cd !$
```

### Dockerfile

Node.jsのパッケージをインストールするためpackage.jsonを用意します。日付処理に便利な[Moment.js](http://momentjs.com/)も追加しました。
 
``` json  ~/docker_apps/ponte-pubsub/package.json
{
  "name": "ponte-pubsub",
  "description": "ponte-pubsub test app",
  "version": "0.0.1",
  "private": true,
  "dependencies": {
    "mqtt": "1.1.0",
    "moment": "2.9.0",
    "moment-timezone": "0.3.0"
  },
  "scripts": {"start": "node app.js"}
}
```

Node.jsのメインプログラムです。3秒間隔でPublishをして同じMQTTクライアントを使ってSubscribeします。

``` js  ~/docker_apps/ponte-pubsub/app.js
var mqtt = require('mqtt'),
    moment = require('moment-timezone');

var options = {
  protocol: "mqtts",
  port: process.env.MQTT_PORT,
  host: process.env.MQTT_HOST,
  username: process.env.MQTT_USERNAME,
  password: process.env.MQTT_PASSWORD,
  clientId: "secure-".concat(Math.floor(65535 * Math.random())),
  rejectUnauthorized : false
};

var client = mqtt.connect(options);

client.on('connect', function() {
  console.log('client connected');
});

client.subscribe('hello');
client.on('message',function(topic, data) {
  console.log("Message on " + topic + ": " + data);
});

setInterval(function() {
  var now = moment().tz("Asia/Tokyo").format("YYYY-MM-DD HH:mm:ssZ");
  client.publish('hello', now);
  console.log('Message Published');
}, 3000);
```

### コンテナを使ってPub/Subのテスト

Dockerfileを作成してイメージをビルドします。

``` bash
$ echo FROM google/nodejs-runtime > Dockerfile
$ docker pull google/nodejs-runtime
$ docker build -t ponte-pubsub .
```

使い捨てのコンテナを起動してPub/Subのテストをします。でMQTTクライアントの設定を環境変数に渡します。

``` bash
$ docker run --rm --name ponte-pubsub \
  -e MQTT_HOST=10.3.0.230  \
  -e MQTT_PORT=8443 \
  -e MQTT_USERNAME=admin \
  -e MQTT_PASSWORD=password \
  ponte-pubsub

> ponte-pubsub@0.0.1 start /app
> node app.js

secure-46408
client connected
Message Published
Message on hello: 2015-03-08 13:47:58+09:00
```

Ponteコンテナのログにもhelloトピックのsubscribeの様子が出力されています。

``` bash
{"name":"ponte","hostname":"17e232fd73f8","pid":1,"service":"MQTT","client":"secure-58538","level":30,"msg":"client connected","time":"2015-03-08T08:23:29.734Z","v":0}
{"name":"ponte","hostname":"17e232fd73f8","pid":1,"service":"MQTT","client":"secure-58538","level":30,"topic":"hello","qos":0,"msg":"subscribed to topic","time":"2015-03-08T08:23:29.742Z","v":0}
{"name":"ponte","hostname":"17e232fd73f8","pid":1,"service":"MQTT","client":"secure-58538","level":30,"topic":"hello","msg":"unsubscribed","time":"2015-03-08T08:23:36.788Z","v":0}
{"name":"ponte","hostname":"17e232fd73f8","pid":1,"service":"MQTT","client":"secure-58538","level":30,"msg":"closed","time":"2015-03-08T08:23:36.793Z","v":0}
```