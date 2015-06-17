title: "IFTTTからBeagleBone BlackのCylon.jsでMQTTを受信する"
date: 2015-02-18 17:05:46
tags:
 - IFTTT
 - Express
 - Nodejs
 - Webhook
 - Yo
 - ngrok
 - Cylonjs
description: IFTTTにYoをして任意のWebhookで通知を受け取ることができました。次はYoをトリガーにしてMQTTブローカーにメッセージをpublishします。手元にあるBeagleBone BlackのCylon.jsのMQTT Driverでsubscribeしてみます。
---

[IFTTTにYoをして](/2015/02/17/express-ifttt-webhook-requestbin/)任意のWebhookで通知を受け取ることができました。次はYoをトリガーにしてMQTTブローカーにメッセージをpublishします。手元にあるBeagleBone BlackのCylon.jsの[MQTT Driver](http://cylonjs.com/documentation/drivers/mqtt/)でsubscribeしてみます。

<!-- more -->

## Webhook

[RequestBin](/2015/02/17/express-ifttt-webhook-requestbin/)にフォワードしたWebhookのコードを修正してMQTTのpublishを追加します。webhookはExpressのミドルウェアなのでapp.jsは普通のExpressとして使えます。webhookに加えて通常のGETリクエストでもMQTTのpublishにブリッジできるようにしました。

```js ~/docker_apps/ifft/app.js
var express = require('express')
  , mqtt = require('mqtt')
  , webhook = require('express-ifttt-webhook');

var client = mqtt.connect('mqtt://admin:password@10.3.0.230:1883');

var app = express();
app.set('port', 8080);

var mqtt_publish = function(json) {
    client.publish('ifttt/bbb',JSON.stringify(json), function(){
        console.log("Message is published");
    });
}

app.get('/', function(req, res) {
    var json = {"message": req.param('message')};
    mqtt_publish(json);
    res.send("Message received");
});

app.use(webhook(function(json,done){
    mqtt_publish(json);
}));

var server = app.listen(app.get('port'), function() {
    console.log('Server listening on port', server.address().port);
});
```

## BeagleBone Black

BeagleBone BlackにはNode.jsを[インストール](/2015/02/09/beagleboneblack-nodejs-npm/)してあります。[Eclipse Orion](http://eclipse.org/orion/)を起動してブラウザでコーディングをしていきます。BeagleBone Blackの操作は[Cylon.js](http://cylonjs.com/)を使います。

### Chromebook

BeagleBone BlackはHP Chromebook 11に[USB-Ethernet接続](/2015/02/15/beagleboneblack-chromebook-usb-internet/)しています。Chromebookから電源供給と[インターネット接続](/2015/02/15/beagleboneblack-chromebook-usb-internet/)をします。

### Cylon.js

Cylon.jsはマルチプラットフォームに対応したデバイス操作用のフレームワークです。[Platforms](http://cylonjs.com/documentation/platforms/)のページを見るとIoT関連で使いたいデバイスやサービス、プロトコルがほとんどあります。

最初にpackage.jsonを記述して必要なnpmモジュールをインストールします。

```js ~/node_modules/orion/.workspace/cylonjs/package.json
{
  "name": "cylon-test",
  "version": "0.0.1",
  "private": true,
  "dependencies": {
    "cylon-beaglebone": "*.*.*",
    "cylon-mqtt": "*.*.*",
    "mqtt": "*.*.*"
  },
  "scripts": {"start": "node app.js"}
}
```

Cylon.jsの[MQTT Driver](http://cylonjs.com/documentation/drivers/mqtt/)のページにあるサンプルを参考にします。MQTTをsubscribeしてメッセージを表示するだけの簡単なプログラムを書きます。

```js ~/node_modules/orion/.workspace/cylonjs/app.js
/*eslint-env node */

var Cylon = require('cylon');

Cylon.robot({
  connections: {
    server: { adaptor: 'mqtt', host: 'mqtt://xxx.xxx.xxx.xxx:1883' },
    beaglebone: { adaptor: 'beaglebone' }
  },
  
  work: function(my) {
    my.server.subscribe('ifttt/bbb');
    my.server.on('message', function (topic, data) {
      console.log(topic + ": " + data);
    });
  }
}).start();
```

## テスト

BeableBone Blackに[インストール](/2015/02/10/beagleboneblack-eclipse-orion/)してあるEclipse OrionのShellを起動して必要なモジュールをインストールします。

``` bash
$ npm install
```

インストールが終了したら`npm start`を実行してMQTTのsubscribeをします。AndroidのYoアプリからIFTTTにYoをすると、BeagleBone Blackでメッセージを受信して標準出力ができました。

```bash
$ npm start 

> cylon-test@0.0.1 start /home/ubuntu/node_modules/orion/.workspace/cylonjs
> node app.js

I, [2015-02-18T06:53:22.388Z]  INFO -- : Initializing connections.
I, [2015-02-18T06:53:24.595Z]  INFO -- : Initializing devices.
I, [2015-02-18T06:53:24.608Z]  INFO -- : Starting connections.
I, [2015-02-18T06:53:24.666Z]  INFO -- : Starting devices.
I, [2015-02-18T06:53:24.669Z]  INFO -- : Working.
ifttt/bbb: {"username":"username","password":"password","title":"Yo","description":"February 18, 2015 at 03:27PM<br>\nvia MA6ATO on Yo","categories":{"string":"http://requestb.in/pb4l5spb"},"tags":[{"string":"IFTTT"},{"string":"Yo"}],"post_status":"publish"}
```