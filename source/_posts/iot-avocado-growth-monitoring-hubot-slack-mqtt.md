title: "IoTでアボカドの発芽促進とカビを防止する - Part3: HubotがMQTTで気温を表示したりLEDライトをつける"
date: 2015-07-14 21:46:47
categories:
 - IoT
tags:
 - IoT
 - Meshblu
 - MQTT
 - Hubot
 - Slack
 - アボカド
description: HubotとSlackのDockerイメージを使ってMQTTのPublishとSubscribeを実装してみます。Hubotのスクリプトはnpmパッケージをrequireしてプログラムを書くことができます。再利用性を考えるとコマンドは外部パッケージにした方がよいですが、カジュアルにscriptsディレクトリにデプロイして使うこともできます。MQTTブローカーに環境データをpublishするRaspberry Pi 2や、MQTTブローカーからメッセージをsubscribeしてLEDライトの電源を制御するBeagleBone Blackについては次回まとめようと思います。
---

[HubotとSlackのDockerイメージ](/2015/07/13/meshblu-compose-hubot-slack/)を使ってMQTTのPublishとSubscribeを実装してみます。Hubotのスクリプトはnpmパッケージをrequireしてプログラムを書くことができます。再利用性を考えるとコマンドは外部パッケージにした方がよいですが、カジュアルにscriptsディレクトリにデプロイして使うこともできます。MQTTブローカーに環境データをpublishするRaspberry Pi 2や、MQTTブローカーからメッセージをsubscribeしてLEDライトの電源を制御するBeagleBone Blackについては次回まとめようと思います。

<!-- more -->

## ユースケース

### 今の気温が知りたい (Subscribe)

Slackからbotに「今の気温は?」や「今の気温は？」に質問すると、MQTTブローカーからsubscribeしている環境データの最新の値を教えてくれます。

### LEDライトをon/offしたい (Publish)

Slackからbotに「ライト付けて」や「ライト消して」と指示すると、MQTTブローカーにpublishして植物育成LEDライトを付けたり消したりしてくれます。


## プロジェクト

前回作ったプロジェクトにいくつか設定を追加しました。リポジトリは[こちら](https://github.com/IDCFChannel/docker-hubot-slack)です。

```bash
$ cd ~/node_apps/docker-hubot-slack
$ tree
├── Dockerfile
├── README.md
├── docker-compose.yml
├── docker-compose.yml.default
├── redis
└── scripts
    ├── hello.coffee
    └── mqtt_meshblu.coffee
```

### Dockerfile

DockerfileではDockerイメージのグローバルにhubotのパッケージをインストールしてあります。[Yeoman](http://yeoman.io/)を使って`/app`ディレクトリにHubotプロジェクトを作成します。Dockerホストの作業ユーザーと同じuidの`docker`ユーザーにスイッチします。Yeomanのジェネレーターが作成した`/app/package.json`に、今回使用する[hubot-slack](https://github.com/slackhq/hubot-slack)と[mqtt](https://github.com/mqttjs/MQTT.js)のパッケージを追加してインストールします。

```bash ~/node_apps/docker-hubot-slack/Dockerfile
FROM iojs:2.3
MAINTAINER Masato Shimizu <ma6ato@gmail.com>

RUN mkdir -p /app
WORKDIR /app

RUN adduser --disabled-password --gecos '' --uid 1000 docker && \
    chown -R docker:docker /app

RUN npm install -g hubot coffee-script yo generator-hubot

USER docker
RUN yes | yo hubot --defaults && \
    npm install --save hubot-slack mqtt
```

## scripts/mqtt_meshblu.coffee

MQTTクライアントは[MQTT.js](https://github.com/mqttjs/MQTT.js)を使います。今回書いた`robot.respond`の正規表現では、botは最低限のコマンドしか理解できません。[Rakuten MA](https://github.com/rakuten-nlp/rakutenma)の形態素解析を使ってもうちょっと賢く応答できるようにしたいです。

```coffeescript ~/node_apps/docker-hubot-slack/scripts/mqtt_meshblu.coffee
# -*- coding: utf-8 -*-

mqtt = require 'mqtt'

opts =
    host: process.env.MQTT_HOST
    username: process.env.ACTION_1_UUID
    password: process.env.ACTION_1_TOKEN
    protocolId: 'MQIsdp'
    protocolVersion: 3

client = mqtt.connect opts
client.subscribe process.env.ACTION_1_UUID

sensor_data = ''

client.on 'message', (topic, message) ->
    payload = JSON.parse message
    sensor_data = payload.data

commands =
    '気温': 'temperature'
    '湿度': 'humidity'
    '気圧': 'pressure'

units =
    '気温': '℃'
    '湿度': '%'
    '気圧': 'hPa'

module.exports = (robot) ->
    robot.respond  /(気温|湿度|気圧)を(おしえて|教えて)$/i, (res) ->
        sensor = res.match[1]
        answer = if sensor_data
            retval = sensor_data[commands[sensor]] + ' ' + units[sensor]
            sensor + 'は ' + retval + ' です。'
        else
            'データが取れません。:imp:'
        res.send answer

    robot.respond /ライトを(つけて|付けて|けして|消して)$/i, (res) ->
        on_off = switch res.match[1]
            when 'つけて', '付けて' then 'on'
            when 'けして', '消して' then 'off'
            else 'on'
        answer = if on_off == 'on' then 'ピカッ。' else 'カチッ。'

        message =
            devices: process.env.ACTION_3_UUID
            payload:
                led: on_off
        client.publish 'message', JSON.stringify(message)

        res.send answer + ':flashlight:'
```

### Subscribeした環境データを表示する

```coffeescript
robot.respond /ライトを(つけて|付けて|けして|消して)$/i, (res) ->
```

Raspberry PiはBME280から計測した環境データをJSON形式でMQTTブローカーにpublishしています。このbotでsubscribeしてメッセージを受信する度にグローバルの`sensor_data`を更新します。botが気温や湿度の質問を受けたときに、受信しているメッセージを返します。

![avocado-temperature.png](/2015/07/14/iot-avocado-growth-monitoring-hubot-slack-mqtt/avocado-temperature.png)


### PublishしてLEDライトのオンオフの指示を出す


```coffeescript
robot.respond /ライト(つけて|付けて|けして|消して)$/i, (res) ->
```

「つけて」の場合は「on」、「けして」の場合は「off」の値をpayloadに入れてメッセージをMQTTブローカーにpublishします。このメッセージはBeagleBone Blackがsubscribeしています。on/offに応じてUSB電源連動タップを接続しているUSBハブのポートの電源を制御します。これだとリモートから本当に植物育成LEDライトが点灯したかわからないのが課題です。Webカメラとか付けたらいいかも。

![avocado-led.png](/2015/07/14/iot-avocado-growth-monitoring-hubot-slack-mqtt/avocado-led.png)
