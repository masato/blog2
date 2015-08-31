title: "myThingsをはじめよう - Part4: 「IDCF」チャンネルのトリガーをHTTPからを使って閾値を超えたらメールする。"
date: 2015-08-27 11:52:59
tags:
---

「IDCF」チャンネルのトリガーは、Raspberry Piと環境センサーが計測した湿度が70%を超えたときなど、閾値を監視してmyThingsのトリガーを作成することができます。

## メッセージの種類

### messageトピック

自作デバイスやWebサービスがMQTTやWebSocketでsubscribeしている場合は、[前回](/2015/08/26/mythings-idcfchannel-messaging/)確認したように、「IDCF」チャンネルを使いMQTT publishやHTTP POST /messagesからメッセージを送信することができます。

### dataトピック

myThingsサーバーは一定間隔でメッセージの受信を確認しています。そのためmyThingsのトリガーとして使う場合は、いったんデータを保存しておく必要があります。

MQTTは`data`トピックに、HTTPは`/data/uuid`に送信していったんMQongoDBにデータを保存します。この後myThingsサーバーが保存されているデータを、一定の間隔で取り出します。myThingsサーバーはデータが取得できたらトリガーの条件がマッチしたと判断して、アクションを実行します。


## trigger-1へMQTT publish data


閾値越えを判断したら、データを`data`トピックに送信します。特定のデバイスにメッセージを送信したときの`message`トピックとは異なり、データを保存する場合は`data`トピックを固定で使います。`devices`キーは、データを保存するuuidなのでtrigger-1自身のuuidになります。

### mosquitto_pubを使ったテスト

mosquitto_pubを使ってある閾値が超えたとき、myThingsのトリガーを発火させるpublishの例です。シェルスクリプトで閾値の監視をする場合はこのまま使えます。

* ホスト: 「IDCF」チャンネルのIPアドレス
* ポート: 1883
* トピック: data
* username: trigger-1のuuid
* password: trigger-1のtoken

```bash
$ mosquitto_pub \
  -h 210.140.162.58  \
  -p 1883 \
  -t data \
  -m '{"devices": ["ffa6934d-f1b3-467f-98b3-766b330d436d"], "payload": "servo on"}' \
  -u ffa6934d-f1b3-467f-98b3-766b330d436d \
  -P d286ba8d \
  -d
```

出力結果

```bash
Received CONNACK
Sending PUBLISH (d0, q0, r0, m1, 'data', ... (76 bytes))
```

### Pythonのプログラム

実際にはRaspberry PiなどにPythonなどのプログラムを書いて、センサーデータの取得と閾値の判断、MQTT publishを書く場合が多いと思います。[myThingsをはじめよう - Part1: はじめてのRaspberry Pi 2セットアップ](/2015/07/16/raspberrypi-2-headless-install-2/)の手順でRaspberry Piとセンサーを準備しておきます。



### trigger-1へ HTTPPOST /data


## myThingsアプリの設定

### 「IDCF」チャンネルのトリガー


### 「Gmail」チャンネルのアクション



「気温が26度をこえたら」と条件をつけます。


