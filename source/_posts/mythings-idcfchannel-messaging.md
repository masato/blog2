title: "myThingsをはじめよう - Part3: MQTTブローカーを使う"
date: 2015-08-26 19:12:36
categories:
 - IoT
tags:
 - IoT
 - REST
 - Meshblu
 - MQTT
 - myThings
 - IDCFクラウド
description: Mesubluは複数のトランポートプロトコルに対応しています。HTTP POSTしたメッセージをMQTTでsubscribeして受信することもできます。IDCFクラウドに構築したMeshbluサービスを起動したあとに、メッセージングとデータ保存のテストをしてみます。
---

[Mesublu](http://meshblu.octoblu.com/)は複数のトランポートプロトコルに対応しています。HTTP POSTしたメッセージをMQTTでsubscribeして受信することもできます。[IDCFクラウドに構築したMeshbluサービス](/2015/08/25/mythings-idcfchannel-setup/)を起動したあとに、メッセージングとデータ保存のテストをしてみます。

<!-- more -->


## MQTTブローカーを使ったメッセージング

MQTTの基本的なメッセージングをテストします。trigger-1とaction-1のuuidを使います。

* メッセージ送信: trigger-1
* メッセージ受信: action-1


### action-1でMQTT subscribe

action-1のuuidを使いメッセージを送信します。「IDCF」チャンネルをインストールしたディレクトリに移動して、コマンドラインツールからtokenとuuidを確認します。

```bash
$ cd ~/iot_apps/meshblu-compose
$ docker-compose run --rm iotutil show -- -k action-1

> iotutil@0.0.1 start /app
> node app.js "show" "-k" "action-1"

┌──────────┬──────────┬──────────────────────────────────────┐
│ keyword  │ token    │ uuid                                 │
├──────────┼──────────┼──────────────────────────────────────┤
│ action-1 │ 8a83d71f │ b61d3398-ac99-4694-9dc4-dd632faf6f6a │
└──────────┴──────────┴──────────────────────────────────────┘
```

action-1からMQTTのメッセージをsubscribeします。[Meshblu](http://meshblu.octoblu.com/)ではsubscribeするトピック名は、メッセージを受信するデバイスのuuidになります。この例ではaction-1のuuidになります。またusernameとpasswordも必須項目です。それぞれデバイスのuuidとtokenを指定します。

* ホスト: 「IDCF」チャンネルのIPアドレス
* ポート: 1883
* username: action-1のuuid
* password: action-1のtoken


```bash
$ mosquitto_sub \
    -h 210.140.162.58  \
    -p 1883  \
    -t b61d3398-ac99-4694-9dc4-dd632faf6f6a  \
    -u b61d3398-ac99-4694-9dc4-dd632faf6f6a  \
    -P 8a83d71f \
    -d
```

出力結果

```bash
Received CONNACK
Received SUBACK
Subscribed (mid: 1): 0
```

### trigger-1からMQTT publish message

trigger-1のuuidを使いメッセージを送信します。tokenとuuidを確認します。

```bash
$ docker-compose run --rm iotutil show -- -k trigger-1

> iotutil@0.0.1 start /app
> node app.js "show" "-k" "trigger-1"

┌───────────┬──────────┬──────────────────────────────────────┐
│ keyword   │ token    │ uuid                                 │
├───────────┼──────────┼──────────────────────────────────────┤
│ trigger-1 │ d286ba8d │ ffa6934d-f1b3-467f-98b3-766b330d436d │
└───────────┴──────────┴──────────────────────────────────────┘
```

Meshbluにメッセージを送信する場合は以下の仕様になります。

* トピック名: message
* 送信先uuid: メッセージのdevicesキーの値
* メッセージ本文: メッセージのpayloadキーの値(JSON、文字列)


新しいシェルを開きます。mosquitto_pubコマンドを使ってpublishします。

```bash
$ mosquitto_pub \
  -h 210.140.162.58  \
  -p 1883 \
  -t message \
  -m '{"devices": ["b61d3398-ac99-4694-9dc4-dd632faf6f6a"], "payload": {"led":"on"}}' \
  -u ffa6934d-f1b3-467f-98b3-766b330d436d \
  -P d286ba8d \
  -d
```

出力結果

```bash
Received CONNACK
Sending PUBLISH (d0, q0, r0, m1, 'message', ... (78 bytes))
```

mosquitto_subコマンドを実行しているシェルにメッセージが届きます。

```bash
{"data":{"led":"on"}}
```

### trigger-1からHTTP POST /messages

MQTT subscriberにHTTP POSTからメッセージを送信することもできます。trigger-1のuuidとtokenは認証情報としてHTTPヘッダに記述します。

* URL: /messages
* Content-Type: application/json
* meshblu_auth_uuid: trigger-1のuuid
* meshblu_auth_token: trigger-1のtoken

先ほどのmosquitto_subコマンドが起動している状態で新しいシェルを開きます。curlでメッセージをPOSTします。

```bash
$ curl -X POST \
  http://210.140.162.58/messages \
  -H "Content-Type: application/json" \
  -d '{"devices": "4dbe71d3-ad32-45e5-99fc-88c225bf11b8", "payload": {"led":"off"}}' \
  --header "meshblu_auth_uuid: ffa6934d-f1b3-467f-98b3-766b330d436d"  \
  --header "meshblu_auth_token: d286ba8d"
```

出力結果

```bash
{"devices":"4dbe71d3-ad32-45e5-99fc-88c225bf11b8","payload":{"led":"off"}}
```

mosquitto_subコマンドを実行しているシェルにメッセージが届きます。

```bash
{"data":{"led":"on"}}
```
