title: "MeshbluにREST APIを使いデバイスを登録する"
date: 2015-03-27 10:44:14
tags:
 - Meshblu
description: MeshbluではデバイスやサービスのノードはUUIDとTOKENで識別されます。whitelistに登録済みのデバイスや、オーナーに指定したデバイス間でのみメッセージの送受信ができます。オーナーを指定しないデバイスはパブリックデバイスになり（デフォルト）どのデバイスからもメッセージが受信ができます。これからメッセージのMQTT-HTTP Bridgeテストをするので準備として必要なデバイスの登録を行います。
---

[Meshblu](https://github.com/octoblu)ではデバイスやサービスのノードはUUIDとTOKENで識別されます。whitelistに登録済みのデバイスや、オーナーに指定したデバイス間でのみメッセージの送受信ができます。オーナーを指定しないデバイスはパブリックデバイスになり（デフォルト）どのデバイスからもメッセージが受信ができます。これからメッセージのMQTT-HTTP Bridgeテストをするので準備として必要なデバイスの登録を行います。

<!-- more -->

## UUID/TOKEN

MeshbluのAPIは操作を実行するデバイスのUUIDとTOKENの指定が必要です。curlの場合は`--header`フラグに`meshblu_auth_uuid: {操作を実行するデバイスのUUID}`と`meshblu_auth_token: {操作を実行するデバイスのTOKEN}`を指定します。URLに含まれる操作対象のデバイスではなく操作を実行するデバイスのUUID/TOKENになるので注意が必要です。APIを実行するデバイスの`username/password`に相当します。


### 任意の文字列で上書き

Meshbluで通常デバイスを登録するとUUIDとTOKENは自動的にランダムで作成されます。

* UUIDの例: 786dd0a0-cd09-11e4-bb2b-d5833b050e1a
* TOKENの例: 672e0fc59ffb545280775037f80926d0343acf4e

UUIDが重複せずTOKENもランダムで作成されるので安全ですが、ローカルで使う場合はちょっと扱いづらいので好きな文字列を指定します。TOKENはRubyを使って6桁の数字でランダムに作成することにします。

``` ruby
$ irb
irb(main):049:0> (0..5).map{rand(0..9)}.join
=> "505478"
```

### 環境変数

3種類のデバイスを登録します。環境変数にさきほど作成したランダムのTOKENと任意のUUIDを指定します。

``` bash ~/.bashrc
export MESHBLU_BROKER=xxx.xxx.xxx.xxx
export MESHBLU_URL=https://$MESHBLU_BROKER

export HTTP_SUB_UUID=http-sub
export HTTP_SUB_TOKEN=505478
export MQTT_PUB_UUID=mqtt-pub
export MQTT_PUB_TOKEN=889346
export TEST_UUID=public-test
export TEST_TOKEN=123
```

* MESHBLU_BROKER: Meshbluのブローカーのドメイン名またはIPアドレス
* MESHBLU_URL: MeshbluのURL、今回はHTTPSで指定します
* HTTP_SUB_UUID: HTTPでsubscribeするデバイスのUUID
* MQTT_PUB_UUID: MQTTでpublishするデバイスのUUID
* TEST_UUID: テスト用のパブリックのデバイスのUUID

## http-subデバイス

http-subデバイスはHTTPでメッセージを受信するデバイスです。curlコマンドでsubscribeするOSXを想定しています。[MeshbluのREADME](https://github.com/octoblu/meshblu#https-rest-api)を読みながら作業を進めます。

### デバイスの登録 (POST)

curlコマンドからHTTPのPOSTメソッドを指定してデバイスを登録します。Dockerコンテナで動いているMeshbluはオレオレ証明書を使っているのでcurlには`--insecure`フラグが必要です。

``` bash
$ curl -X POST \
  "$MESHBLU_URL/devices" \
  --insecure \
  -d "name=$HTTP_SUB_UUID&uuid=$HTTP_SUB_UUID&token=$HTTP_SUB_TOKEN"
{"geo":null,"ipAddress":"172.17.42.1","name":"http-sub","online":false,"timestamp":"2015-03-27T00:47:56.940Z","uuid":"http-sub","token":"505478"}
```


### claimdeviceでデバイスの所有権を取得する

後ほど実行するようにデバイスにwhitelistとblacklistを設定して操作の権限を制御します。whitelistに指定したデバイスからのみ操作が可能になります。逆にblacklistに指定されたデバイスからは操作ができなくなります。デバイスのUUIDを文字列または配列で指定します。デフォルトではどの操作権限もwhitelistとblacklistもありません。デバイスはパブリックであり他のデバイスからすべての操作が可能です。ローカルのブローカーでもこれだと不都合があるので役割に応じて権限を指定します。

* discoverWhitelist: リストできる
* discoverBlacklist: リストできない
* sendWhitelist: メッセージを送信できる
* sendBlacklist: メッセージを送信できない
* receiveWhitelist: メッセージを受信できる
* receiveBlacklist: メッセージを受信できない
* updateWhitelist: 情報を更新できる
* updateBlacklist: 情報を更新できない

まずはclaimdeviceを実行してデバイス自身がオーナーになります。デバイス自信がオーナーになると自分以外のデバイスからは見えなくなります。

``` bash
$ curl -X PUT \
  "$MESHBLU_URL/claimdevice/$HTTP_SUB_UUID" \
  --insecure  \
  --header "meshblu_auth_uuid: $HTTP_SUB_UUID" \
  --header "meshblu_auth_token: $HTTP_SUB_TOKEN"
{"discoverWhitelist":["http-sub"],"geo":null,"ipAddress":"172.17.42.1","name":"http-pub","online":false,"owner":"http-sub","timestamp":"2015-03-27T00:49:48.447Z","uuid":"http-sub"}
```

## mqtt-pubデバイス

mqtt-pubデバイスはMQTTのpublishをするデバイスです。MosquittoをインストールしたRaspberry Piを想定しています。http-subデバイスと同様にデバイスを作成したあと`claimdevice`を実行してデバイス自身がオーナーになります。

``` bash
$ curl -X POST \
  "$MESHBLU_URL/devices" \
  --insecure \
  -d "name=MQTT_PUB_UUID&uuid=$MQTT_PUB_UUID&token=$MQTT_PUB_TOKEN"
{"geo":null,"ipAddress":"172.17.42.1","name":"mqtt-pub","online":false,"timestamp":"2015-03-27T00:50:31.614Z","uuid":"mqtt-pub","token":"889346"}
$ curl -X PUT \
  "$MESHBLU_URL/claimdevice/$MQTT_PUB_UUID" \
  --insecure  \
  --header "meshblu_auth_uuid: $MQTT_PUB_UUID" \
  --header "meshblu_auth_token: $MQTT_PUB_TOKEN"
{"discoverWhitelist":["mqtt-pub"],"geo":null,"ipAddress":"172.17.42.1","name":"mqtt-pub","online":false,"owner":"mqtt-pub","timestamp":"2015-03-27T00:54:11.676Z","uuid":"mqtt-pub"}
```

### デバイスの更新 (PUT)

mqtt-pubデバイスからhttp-subデバイスにメッセージを送れるようにhttp-subデバイスのwhitelistを更新します。他のデバイスがメッセージを送信したりデバイス情報をみることができるようにするには、対象デバイス(http-sub)のdiscoverWhitelistとsendWhitelistに操作を許可したいデバイス(mqtt-pub)のUUIDを指定して更新します。

`from`から始まるJSONの戻り値のように、更新前のdiscoverWhitelistは`"discoverWhitelist":["http-sub"]`になっています。mqtt-pubデバイスだけ指定すると自分(http-subデバイス)がwhitelistからいなくなります。ただし`"owner":"http-sub"`に指定されオーナーになっているので自分自身へはすべての操作が許可されています。

``` bash
$ curl -X PUT \
  -d "discoverWhitelist=[$MQTT_PUB_UUID]&sendWhitelist=[$MQTT_PUB_UUID]" \
  "$MESHBLU_URL/devices/$HTTP_SUB_UUID" \
  --insecure  \
  --header "meshblu_auth_uuid: $HTTP_SUB_UUID" \
  --header "meshblu_auth_token: $HTTP_SUB_TOKEN"
{"discoverWhitelist":"[mqtt-pub]","geo":null,"ipAddress":"172.17.42.1","name":"http-sub","online":false,"owner":"http-sub","sendWhitelist":"[mqtt-pub]","timestamp":"2015-03-27T00:58:16.467Z","uuid":"http-sub","fromUuid":"http-sub","from":{"_id":"5519eebc6c04bc0a002d59ae","discoverWhitelist":["http-sub"],"geo":null,"ipAddress":"172.17.42.1","name":"http-sub","online":false,"owner":"http-sub","timestamp":"2015-03-27T00:52:34.978Z","uuid":"http-sub"}}
```

## public-testデバイス

テスト用にパブリックデバイスを作成します。`claimdevice`しないのでどのデバイスからも操作が可能です。

``` bash
$ curl -X POST \
  "$MESHBLU_URL/devices" \
  --insecure \
  -d "name=$TEST_UUID&uuid=$TEST_UUID&token=$TEST_TOKEN"
{"geo":null,"ipAddress":"172.17.42.1","name":"public-test","online":false,"timestamp":"2015-03-27T01:00:36.966Z","uuid":"public-test","token":"123"}
```

## デバイスのリスト (GET)

APIを実行するデバイスのUUID/TOKENを`--header`フラグに指定します。URLに含まれるデバイスIDがリストできるか確認します。

mqtt-pubデバイスからhttp-subデバイスはリストできます。

``` bash
$ curl -X GET \
  "$MESHBLU_URL/devices/$HTTP_SUB_UUID" \
  --insecure \
  --header "meshblu_auth_uuid: $MQTT_PUB_UUID" \
  --header "meshblu_auth_token: $MQTT_PUB_TOKEN"
{"devices":[{"geo":null,"ipAddress":"172.17.42.1","name":"http-sub","online":false,"timestamp":"2015-03-27T00:58:16.467Z","uuid":"http-sub"}]}
```

http-subデバイスからmqtt-pubデバイスはリストできません。

``` bash
$ curl -X GET \
  "$MESHBLU_URL/devices/$MQTT_PUB_UUID" \
  --insecure \
  --header "meshblu_auth_uuid: $HTTP_SUB_UUID" \
  --header "meshblu_auth_token: $HTTP_SUB_TOKEN"
{"devices":[]}
```

public-testはパブリックデバイスなので他のデバイス(http-subデバイス)からリストできます。http-subデバイスからは自分自身とパブリックのpublic-testデバイスが見えます。

``` bash
$ curl -X GET \
  "$MESHBLU_URL/devices" \
  --insecure \
  --header "meshblu_auth_uuid: $HTTP_SUB_UUID" \
  --header "meshblu_auth_token: $HTTP_SUB_TOKEN"
{"devices":[{"geo":null,"ipAddress":"172.17.42.1","name":"public-test","online":false,"timestamp":"2015-03-27T01:07:40.758Z","uuid":"public-test"},{"discoverWhitelist":"[mqtt-pub]","geo":null,"ipAddress":"172.17.42.1","name":"http-sub","online":false,"owner":"http-sub","sendWhitelist":"[mqtt-pub]","timestamp":"2015-03-27T00:58:16.467Z","uuid":"http-sub"}]}
```

public-testデバイスは他のどのデバイスの`discoverWhitelist`にも登録されていないので自分自身のデバイスしか見えません。

``` bash
$ curl -X GET \
  "$MESHBLU_URL/devices" \
  --insecure \
  --header "meshblu_auth_uuid: $TEST_UUID" \
  --header "meshblu_auth_token: $TEST_TOKEN"
{"devices":[{"geo":null,"ipAddress":"172.17.42.1","name":"public-test","online":false,"timestamp":"2015-03-27T01:07:40.758Z","uuid":"public-test"}]}
```


## デバイスの削除 (DELETE)

不要になったデバイスはDELETEメソッドで削除できます。http-subデバイスからテスト用のpublic-testデバイスを削除します。

``` bash
$ curl -X DELETE \
  "$MESHBLU_URL/devices/$TEST_UUID" \
  --insecure \
  --header "meshblu_auth_uuid: $HTTP_SUB_UUID" \
  --header "meshblu_auth_token: $HTTP_SUB_TOKEN"
{"uuid":"public-test"}
```

削除したデバイスをリストすると`Devices not found`になります。

``` bash
$ curl -X GET \
  "$MESHBLU_URL/devices/$TEST_UUID" \
  --insecure \
  --header "meshblu_auth_uuid: $HTTP_SUB_UUID" \
  --header "meshblu_auth_token: $HTTP_SUB_TOKEN"
{"message":"Devices not found","code":404}
```