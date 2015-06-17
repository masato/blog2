title: "Meshbluを使いMQTTとHTTPをブリッジしてRaspberry Piでメッセージを送受信する"
date: 2015-03-28 06:20:14
tags:
 - Meshblu
 - MQTT
 - Mosquitto
 - RaspberryPi
description: Meshblu登録したデバイスを使いMQTTとHTTP間で相互にメッセージの送受信を試してみます。複数のプロトコルの違いをMeshbluのブローカーが吸収してくれるのでより多くのサービスやデバイスがメッセージを交換できるようになります。
---

[Meshblu登録したデバイス](/2015/03/27/meshblu-register-devices)を使いMQTTとHTTP間で相互にメッセージの送受信を試してみます。複数のプロトコルの違いをMeshbluのブローカーが吸収してくれるのでより多くのサービスやデバイスがメッセージを交換できるようになります。

<!-- more -->

## 環境変数

デバイスの登録と利用に使う環境変数です。`~/.bashrc`などに書いておきます。

````bash ~/.bashrc
export MESHBLU_BROKER=xxx.xxx.xxx.xxx
export MESHBLU_URL=https://$MESHBLU_BROKER
export HTTP_SUB_UUID=http-sub
export HTTP_SUB_TOKEN=505478
export MQTT_PUB_UUID=mqtt-pub
export MQTT_PUB_TOKEN=889346
```


## デバイスのメッセージ送受信権限

デバイスのownerとwhitelistの権限は[前回](/2015/03/27/meshblu-register-devices)設定した状態からはじめます。説明上`http-sub`、`mqtt-pub`というUUIDを付けていますが、どちらも自分がデバイスのオーナーなので自分自身に対してはメッセージの送受信が可能です。


### http-subデバイス

`http-sub`デバイスは以下のような設定にしています。OSX上でメッセージを受信するときに使う想定です。

* `"protocol":"http"` -> HTTPでメッセージを受信できる
* `"sendWhitelist":"[mqtt-pub]"` -> `mqtt-pub`デバイスからメッセージを受信できる
* "owner":"http-sub" -> 自分がオーナーなので`http-sub`デバイスにメッセージを送受信できる

``` bash
$ curl -X GET \
  "$MESHBLU_URL/devices/$HTTP_SUB_UUID" \
  --insecure \
  --header "meshblu_auth_uuid: $HTTP_SUB_UUID" \
  --header "meshblu_auth_token: $HTTP_SUB_TOKEN"
{"devices":[{"discoverWhitelist":"[mqtt-pub]","geo":null,"ipAddress":"172.17.42.1","name":"http-sub","online":false,"owner":"http-sub","protocol":"http","sendWhitelist":"[mqtt-pub]","timestamp":"2015-03-28T20:02:54.875Z","uuid":"http-sub"}]}
```

### mqtt-subデバイス

`mqtt-pub`デバイスは以下のような設定にしています。Raspberry Pi上でメッセージを受信するときに使う想定です。

* `"protocol":"mqtt"` -> MQTTでメッセージを受信できる
* "owner":"mqtt-pub" -> 自分がオーナーなので`mqtt-pub`デバイスにメッセージを送受信できる

``` bash
$ curl -X GET \
  "$MESHBLU_URL/devices/$MQTT_PUB_UUID" \
  --insecure \
  --header "meshblu_auth_uuid: $MQTT_PUB_UUID" \
  --header "meshblu_auth_token: $MQTT_PUB_TOKEN"
{"devices":[{"discoverWhitelist":["mqtt-pub"],"geo":null,"ipAddress":"172.17.42.1","name":"mqtt-pub","online":true,"onlineSince":"2015-03-28T01:37:48.875Z","owner":"mqtt-pub","protocol":"mqtt","timestamp":"2015-03-28T20:22:25.885Z","uuid":"mqtt-pub"}]}
```

## デバイスのUUID/TOKENを使うマシン

用意した2つのデバイスのUUID/TOKENはそれぞれ以下のマシンからメッセージを送受信するときに使います。

* `mqtt-pub`デバイスのUUID/TOKEN: Raspberry Pi
* `http-sub`デバイスのUUID/TOKEN: OSX

デバイス情報には`geo`や`ipAddress`のキーがありますが、今のところ物理的なマシンに結びついてはいないのでUUID/TOKENはどのマシンからも利用することができるようです。

## MQTT(mqtt-pub) > HTTP(http-sub)

UUIDをauthに使います。`mosquitto_sub`や`mosquitto_pub`コマンドからメッセージ送信または受信すると、デバイス情報のprotocolがmqttに更新されます。

mosquitto_subコマンドを実行してみます。

``` bash
$ mosquitto_sub \
  -h $MESHBLU_BROKER  \
  -p 1883 \
  -t $HTTP_SUB_UUID \
  -u $HTTP_SUB_UUID \
  -P $HTTP_SUB_TOKEN \
  -d
Client mosqsub/1830-minion1.cs sending CONNECT
Client mosqsub/1830-minion1.cs received CONNACK
Client mosqsub/1830-minion1.cs sending SUBSCRIBE (Mid: 1, Topic: http-sub, QoS: 0)
Client mosqsub/1830-minion1.cs received SUBACK
Subscribed (mid: 1): 0
```

デバイス情報を確認するとprotocolがmqttになってしまいました。

``` bash
$ curl -X GET \
  "$MESHBLU_URL/devices/$HTTP_SUB_UUID" \
  --insecure \
  --header "meshblu_auth_uuid: $HTTP_SUB_UUID" \
  --header "meshblu_auth_token: $HTTP_SUB_TOKEN"
{"devices":[{"discoverWhitelist":"[mqtt-pub]","geo":null,"ipAddress":"172.17.42.1","name":"http-sub","online":false,"owner":"http-sub","protocol":"mqtt","sendWhitelist":"[mqtt-pub]","timestamp":"2015-03-28T20:02:54.875Z","uuid":"http-sub"}]}
```

HTTPでメッセージを受信する場合はprotocolをhttpに変更する必要があります。デフォルトだとprotocolはデバイス情報のJSONに含まれていません。UUIDで一度もMQTTを使わなければprotocolの更新は不要ですが念のため行います。

``` bash
$ curl -X PUT \
  "$MESHBLU_URL/devices/$HTTP_SUB_UUID" \
  --insecure \
  -d "token=$HTTP_SUB_TOKEN&protocol=http" \
  --header "meshblu_auth_uuid: $HTTP_SUB_UUID" \
  --header "meshblu_auth_token: $HTTP_SUB_TOKEN"
{"discoverWhitelist":"[mqtt-pub]","geo":null,"ipAddress":"172.17.42.1","name":"http-sub","online":false,"owner":"http-sub","protocol":"http","sendWhitelist":"[mqtt-pub]","timestamp":"2015-03-28T01:26:31.531Z","uuid":"http-sub","fromUuid":"http-sub","from":{"_id":"5519eebc6c04bc0a002d59ae","discoverWhitelist":"[mqtt-pub]","geo":null,"ipAddress":"172.17.42.1","name":"http-sub","online":false,"owner":"http-sub","sendWhitelist":"[mqtt-pub]","timestamp":"2015-03-28T00:58:16.467Z","uuid":"http-sub"}}
```

OSX上で`http-sub`デバイスをauth情報にしてcurlからメッセージをsubscribeします。publishされる前にsubscribeしている必要があります。

``` bash
$ curl -X GET \
  "$MESHBLU_URL/subscribe" \
  --insecure \
  --header "meshblu_auth_uuid: $HTTP_SUB_UUID" \
  --header "meshblu_auth_token: $HTTP_SUB_TOKEN"
```

Raspberry Pi上で`mqtt-pub`デバイスをauth情報にしてMosquitto Clientsからメッセージをpublishします。

``` bash
$ mosquitto_pub \
  -h $MESHBLU_BROKER  \
  -p 1883 \
  -t message \
  -m '{"devices": ["'"$HTTP_SUB_UUID"'"], "payload": {"red":"on"}}' \
  -u $MQTT_PUB_UUID \
  -P $MQTT_PUB_TOKEN \
  -d
Client mosqpub/30699-raspberry sending CONNECT
Client mosqpub/30699-raspberry received CONNACK
Client mosqpub/30699-raspberry sending PUBLISH (d0, q0, r0, m1, 'message', ... (50 bytes))
Client mosqpub/30699-raspberry sending DISCONNECT
```

OSX上でsubscribeしているシェルにメッセージが届きました。

``` bash
{"devices":["http-sub"],"payload":{"red":"on"},"fromUuid":"mqtt-pub"},
```

## HTTP(http-sub) > HTTP(http-sub)

OSX上で`http-sub`デバイスをauthに情報にしてcurlからメッセージをsubscribeします。

``` bash
$ curl -X GET \
  "$MESHBLU_URL/subscribe" \
  --insecure \
  --header "meshblu_auth_uuid: $HTTP_SUB_UUID" \
  --header "meshblu_auth_token: $HTTP_SUB_TOKEN"
```

`http-sub`デバイスから`http-sub`デバイスへは自分がオーナーなのでメッセージを送信できます。

``` bash
$ curl -X POST \
  "$MESHBLU_URL/messages" \
  --insecure \
  -d '{"devices": ["'"$HTTP_SUB_UUID"'"], "payload": {"red":"on"}}' \
  --header "meshblu_auth_uuid: $HTTP_SUB_UUID" \
  --header "meshblu_auth_token: $HTTP_SUB_TOKEN"
{"devices":["http-sub"],"payload":{"red":"on"}}
```

OSX上でsubscribeしているシェルにメッセージが届きました。

``` bash
{"devices":["http-sub"],"payload":{"red":"on"},"fromUuid":"http-sub"},
```

## MQTT(mqtt-pub) > MQTT(mqtt-pub)

Raspberry Pi上で`mqtt-pub`デバイスをauth情報にしてsubscribeします。

``` bash
$ mosquitto_sub \
  -h $MESHBLU_BROKER  \
  -p 1883 \
  -t $MQTT_PUB_UUID \
  -u $MQTT_PUB_UUID \
  -P $MQTT_PUB_TOKEN \
  -d
Client mosqsub/26961-raspberry sending CONNECT
Client mosqsub/26961-raspberry received CONNACK
Client mosqsub/26961-raspberry sending SUBSCRIBE (Mid: 1, Topic: mqtt-pub, QoS: 0)
Client mosqsub/26961-raspberry received SUBACK
Subscribed (mid: 1): 0
```

OSX上で同様にpublishします。

``` bash
$ mosquitto_pub \
  -h $MESHBLU_BROKER  \
  -p 1883 \
  -t message \
  -m '{"devices": ["'"$MQTT_PUB_UUID"'"], "payload": {"red":"on"}}' \
  -u $MQTT_PUB_UUID \
  -P $MQTT_PUB_TOKEN \
  -d
Client mosqpub/16284-minion1.c sending CONNECT
Client mosqpub/16284-minion1.c received CONNACK
Client mosqpub/16284-minion1.c sending PUBLISH (d0, q0, r0, m1, 'message', ... (50 bytes))
Client mosqpub/16284-minion1.c sending DISCONNECT
```

Raspberry Pi上でsubscribeしているシェルにメッセージが届きました。

``` bash
{"topic":"message","data":{"devices":["mqtt-pub"],"payload":{"red":"on"},"fromUuid":"mqtt-pub"}}
```

## HTTP(http-sub) > MQTT(mqtt-pub)

Raspberry Pi上で`mqtt-pub`デバイスをauth情報にしてsubscribeします。

``` bash
$ mosquitto_sub \
  -h localhost  \
  -p 1883 \
  -t $MQTT_PUB_UUID \
  -u $MQTT_PUB_UUID \
  -P $MQTT_PUB_TOKEN \
  -d
```

OSX上で`http-sub`デバイスをauthに情報にしてcurlからメッセージをpublishします。

``` bash
$ curl -X POST \
  "$MESHBLU_URL/messages" \
  --insecure \
  -d '{"devices": ["'"$MQTT_PUB_UUID"'"], "payload": {"red":"on"}}' \
  --header "meshblu_auth_uuid: $HTTP_SUB_UUID" \
  --header "meshblu_auth_token: $HTTP_SUB_TOKEN"
{"devices":["mqtt-pub"],"payload":{"red":"on"}}
```

Raspberry Pi上でsubscribeしているシェルにメッセージが届きました。

``` bash
{"topic":"message","data":{"devices":["mqtt-pub"],"payload":{"red":"on"},"fromUuid":"http-sub"}}
```
