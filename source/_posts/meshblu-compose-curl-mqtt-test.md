title: "MeshbluでオープンソースのIoTをはじめよう - Part5: RESTとMQTTのメッセージングとデータ保存"
date: 2015-07-17 19:12:36
categories:
 - IoT
tags:
 - IoT
 - REST
 - Meshblu
 - MQTT
 - curl
 - IDCFクラウド
description: Mesubluは複数のトランポートプロトコルに対応しています。REST APIでPOSTしたメッセージをMQTTでsubscribeして受信することができます。クラウド上のDockerホストにMeshbluコンテナを起動したあと、localhostでメッセージングとデータ保存のテストをしてみます。
---

[Mesublu](https://developer.octoblu.com/)は複数のトランポートプロトコルに対応しています。REST APIでPOSTしたメッセージをMQTTでsubscribeして受信することができます。クラウド上のDockerホストにMeshbluコンテナを起動したあと、localhostでメッセージングとデータ保存のテストをしてみます。

<!-- more -->

## Meshbluのインストール

MeshbluとRedis、MongoDB、OpenResty、CLIツールなど一式をDocker Composeで起動します。リポジトリは[こちら](https://github.com/IDCFChannel/meshblu-compose)です。

```bash
$ sudo apt-get update && sudo apt-get install -y git
$ mkdir ~/iot_apps && cd ~/iot_apps
$ git clone https://github.com/IDCFChannel/meshblu-compose
$ cd meshblu-compose
```

インストールスプリクトを実行するとMeshbluのコンテナが起動します。

```bash
$ ./bootstrap.sh
...
      Name             Command             State              Ports
-------------------------------------------------------------------------
meshblucompose_m   npm start          Up                 0.0.0.0:1883->18
eshblu_1                                                 83/tcp, 80/tcp
meshblucompose_m   /entrypoint.sh     Up                 27017/tcp
ongo_1             mongod
meshblucompose_o   nginx -c /etc/ng   Up                 0.0.0.0:443->443
penresty_1         inx/nginx.conf                        /tcp, 0.0.0.0:80
                                                         ->80/tcp
meshblucompose_r   /entrypoint.sh     Up                 6379/tcp
edis_1             redis-server
```


### curlとMosquitto

RESTはcurlを使い、MQTTは[Mosquitto](http://mosquitto.org/)をクライアントに使ってテストをします。

```bash
$ sudo apt-get install curl mosquitto-clients
```


## デバイスの準備

MeshbluではREST APIやMQTTを使ってメッセージをpubsubするクライアントを「デバイス」として管理します。

### デバイスの登録

インストール直後はデバイスは登録されていません。最初にdocker-compose.ymlに付属しているCLIを使いプリセットのデバイスを登録します。

```bash
$ cd ~/iot_apps/meshblu-compose
$ docker-compose run --rm iotutil register

> iotutil@0.0.1 start /app
> node app.js "register"

┌─────────┬────────────────────┬──────────────────────────────────────┐
│ keyword │ meshblu_auth_token │ meshblu_auth_uuid                    │
├─────────┼────────────────────┼──────────────────────────────────────┤
│ owner   │ 8ec3992b           │ ed311cd7-dc09-459b-9aaf-7d1c08186bdd │
└─────────┴────────────────────┴──────────────────────────────────────┘
```

コマンドを実行すると、`trigger-[1-5]`と`action-[1-5]`とownerをあわせて合計11個のデバイスが登録されます。標準出力されるownerのデバイスは作成した11個のデバイスすべてにメッセージを送信することができます。デフォルトでは、`trigger-[1-5]`から`action-[1-5]`へ末尾が同じ数字の組み合わせのメッセージ送信を許可しています。

### デバイスの一覧

登録したデバイスは以下のコマンドで確認することができます。uuidとtokenはMeshbluのブローカーに接続するための認証情報としても利用します。

```bash
$ docker-compose run --rm iotutil list

> iotutil@0.0.1 start /app
> node app.js "list"

┌───────────┬──────────┬──────────────────────────────────────┐
│ keyword   │ token    │ uuid                                 │
├───────────┼──────────┼──────────────────────────────────────┤
│ trigger-1 │ 6e8e63fe │ 71a24381-02e2-47f3-bfa6-8174545f2261 │
├───────────┼──────────┼──────────────────────────────────────┤
│ trigger-2 │ 85a97be4 │ 561bc67b-0c0c-4ad4-a6ec-f1a6163f02b0 │
├───────────┼──────────┼──────────────────────────────────────┤
│ trigger-3 │ ede956b8 │ 6a21c6b7-2a1c-40e3-a26c-96b52f48c586 │
├───────────┼──────────┼──────────────────────────────────────┤
│ trigger-4 │ 4e120995 │ 9f109642-efa9-4973-9bb4-4bc48987e5ec │
├───────────┼──────────┼──────────────────────────────────────┤
│ trigger-5 │ 190ade78 │ 12cdc02d-f1ac-43ef-936a-a44565a58f2f │
├───────────┼──────────┼──────────────────────────────────────┤
│ action-1  │ a02222d2 │ 4dbe71d3-ad32-45e5-99fc-88c225bf11b8 │
├───────────┼──────────┼──────────────────────────────────────┤
│ action-2  │ 4cb06821 │ 6244ee9c-203c-478a-992b-5fe1c5f782cb │
├───────────┼──────────┼──────────────────────────────────────┤
│ action-3  │ 54c454e5 │ b3b4cee4-2f19-4c05-904e-1fa139f531bc │
├───────────┼──────────┼──────────────────────────────────────┤
│ action-4  │ 6b80470a │ c2986c27-e751-47d1-b655-d1750e3ecb71 │
├───────────┼──────────┼──────────────────────────────────────┤
│ action-5  │ 3aefc667 │ fd9f1a40-fa99-44f4-a405-49f360c175aa │
└───────────┴──────────┴──────────────────────────────────────┘
```


## Mesubluの状態確認

### APIの仕様

MeshbluのREST APIやMQTTの使い方は[Documentation](http://meshblu-http.readme.io/docs)のサイトに詳しく書いてあります。

### SSL通信のinsecureフラグについて

Docker ComposeでMeshbuを起動するとフロントのリバースプロキシとしてOpenRestyが起動しています。Dockerホストの80と443にマップされています。


```bash
$ docker-compose  ps openresty
      Name             Command             State              Ports
-------------------------------------------------------------------------
meshblucompose_o   nginx -c /etc/ng   Up                 0.0.0.0:443->443
penresty_1         inx/nginx.conf                        /tcp, 0.0.0.0:80
                                                         ->80/tcp
```

http(80)はすべてhttps(443)にリダイレクトしています。`./bootstrap.sh`を実行するときに自己署名のSSL証明書を作成しているので、curlコマンドからHTTPSを使う場合は`--insecure`オプションが必要です。

```bash ~/iot_apps/meshblu-compose/nginx/conf/nginx.conf
http {
    server {
        listen 80;
        server_name localhost;
        underscores_in_headers on;
        return 301 https://$host$request_uri;
    }
```

### GET /status

curlからMeshbluが起動しているか`status`を確認します。

```bash
$ curl -X GET \
  "https://localhost/status" \
  --insecure \
  --header "meshblu_auth_uuid: ed311cd7-dc09-459b-9aaf-7d1c08186bdd" \
  --header "meshblu_auth_token: 8ec3992b"
```

結果

```json
{"meshblu":"online"}
```

## メッセージを送信して、すぐに受信する

MQTTブローカーを使った一番基本的なメッセージングから試してみます。テスト用のデバイスとして`trigger-1`と`action-1`を使います。

* メッセージ送信: trigger-1
* メッセージ受信: action-1

メッセージを送信するデバイスはtrigger-1です。tokenとuuidを確認します。

```bash
$ docker-compose run --rm iotutil show -- -k trigger-1

> iotutil@0.0.1 start /app
> node app.js "show" "-k" "trigger-1"

┌───────────┬──────────┬──────────────────────────────────────┐
│ keyword   │ token    │ uuid                                 │
├───────────┼──────────┼──────────────────────────────────────┤
│ trigger-1 │ 6e8e63fe │ 71a24381-02e2-47f3-bfa6-8174545f2261 │
└───────────┴──────────┴──────────────────────────────────────┘
```

メッセージを受信するデバイスはaction-1です。tokenとuuidを確認します。

```bash
$ docker-compose run --rm iotutil show -- -k action-1

> iotutil@0.0.1 start /app
> node app.js "show" "-k" "action-1"

┌──────────┬──────────┬──────────────────────────────────────┐
│ keyword  │ token    │ uuid                                 │
├──────────┼──────────┼──────────────────────────────────────┤
│ action-1 │ a02222d2 │ 4dbe71d3-ad32-45e5-99fc-88c225bf11b8 │
└──────────┴──────────┴──────────────────────────────────────┘
```


### MQTT subscribe uuid

最初にmosquitto_subコマンドを使い、action-1がMQTTのメッセージをsubscribeします。Meshbluではメッセージをsubscribeするトピック名はメッセージを受信するデバイスのuuidになります。またusernameとpasswordも必須項目です。それぞれuuidとtokenを指定します。

```bash
$ mosquitto_sub \
    -h localhost  \
    -p 1883  \
    -t 4dbe71d3-ad32-45e5-99fc-88c225bf11b8  \
    -u 4dbe71d3-ad32-45e5-99fc-88c225bf11b8  \
    -P a02222d2 \
    -d
```

### MQTT publish message

trigger-1からaction-1にメッセージを送信します。mosquitto_subと同様にusernameとpasswordはtrigger-1のuuidとtokenを指定します。トピック名は`message`が固定なので注意が必要です。送信先はJSON形式のメッセージの中に`devices`キーの値として指定します。action-1にメッセージを送信するので、action-1のuuidの`4dbe71d3-ad32-45e5-99fc-88c225bf11b8`を記述します。メッセージの本文は`payload`キーに指定します。この例ではJSON形式で記述していますが、文字列も使えます。また`payload`キーと値がなくても空でメッセージを送信することもできます。


新しいシェルを開き、mosquitto_pubコマンドからpublishします。

```bash
$ mosquitto_pub \
  -h localhost  \
  -p 1883 \
  -t message \
  -m '{"devices": ["4dbe71d3-ad32-45e5-99fc-88c225bf11b8"], "payload": {"led":"on"}}' \
  -u 71a24381-02e2-47f3-bfa6-8174545f2261 \
  -P 6e8e63fe \
  -d
```

mosquitto_subコマンドを実行しているシェルにメッセージが届きました。

```bash
{"data":{"led":"on"}}
```

### HTTP POST /messages

MQTTのsubscriberにREST APIからメッセージをPOSTしてみます。RESTのクライアントはcurlコマンドを使います。MQTTのpublishと同様にURLは`/messages`が固定です。トピック名と違いURLの方は`/messages`と複数形になっています。メッセージはJSON形式なので`Content-Type: application/json`が必要です。また、trigger-1のuuidとtokenは認証情報としてHTTPヘッダに記述します。

先ほどのmosquitto_subコマンドが起動している状態で、新しいシェルを開きcurlコマンドからメッセージをPOSTします。

```bash
$ curl -X POST \
  "https://localhost/messages" \
  --insecure  \
  -H "Content-Type: application/json" \
  -d '{"devices": "4dbe71d3-ad32-45e5-99fc-88c225bf11b8", "payload": {"led":"on"}}' \
  --header "meshblu_auth_uuid: 71a24381-02e2-47f3-bfa6-8174545f2261"  \
  --header "meshblu_auth_token: 6e8e63fe"
```

メッセージのPOSTに成功すると以下のメッセージが表示されます。

```bash
{"devices":"4dbe71d3-ad32-45e5-99fc-88c225bf11b8","payload":{"led":"on"}}
```

mosquitto_subコマンドを実行しているシェルにメッセージが届きました。

```bash
{"data":{"led":"on"}}
```

## データを保存して、あとで取り出す

Raspberry Piと環境センサーが計測した湿度が70%を超えたときなど、特定の閾値をデータとして送信したい場合があります。

データの送信先がMQTTやWebSocketでsubscribeしている場合は、上記で確認したようにMQTT publishやHTTP POST /messagesからメッセージとして送信できます。ただし、データの送信先がHTTPの場合はメッセージを同期で受信するのが困難になります。

MQTTの場合は`data`トピックに、HTTPの場合は`/data/uuid`に送信して一度MongoDBにデータを保存します。データの送信先はMongoDBにレンジクエリのAPIを使い保存されているデータをポーリングして取り出しします。

### MQTT publish data

trigger-1から閾値越えのデータを`data`トピックに送信してみます。特定のデバイスにメッセージを送信したときの`message`トピックとは異なり、データを保存する場合は`data`トピックを固定で使います。`devices`キーは、データを保存するuuidなのでtrigger-1自身のuuidになるので注意が必要です。

```bash
$ mosquitto_pub \
  -h localhost  \
  -p 1883 \
  -t data \
  -m '{"devices": ["71a24381-02e2-47f3-bfa6-8174545f2261"], "payload": "trigger on"}' \
  -u 71a24381-02e2-47f3-bfa6-8174545f2261 \
  -P 6e8e63fe \
  -d
```

### MongoDBのデータ

MondoDBコンテナに入り保存されたデータを確認します。MondoDBのデータベース名は`skynet`です。

```bash
$ cd ~/iot_apps/meshblu_compose
$ docker exec -it meshblucompose_mongo_1 mongo skynet

trigger-1がmosquitto_pubしたデータを確認します。

```bash
> db.data.find().sort({ $natural: -1 }).limit(1);
{ "_id" : ObjectId("55b212f5ac81f6100042b353"), "devices" : [ "71a24381-02e2-47f3-bfa6-8174545f2261" ], "payload" : "trigger on", "fromUuid" : "71a24381-02e2-47f3-bfa6-8174545f2261", "uuid" : "71a24381-02e2-47f3-bfa6-8174545f2261", "timestamp" : "2015-07-24T10:27:01.766Z" }
```

### HTTP GET /data/uuid

このレンジクエリを実行するデバイスのuuidとtokenはすべてのデバイスにアクセスができるownerデバイスのuuidとtokenを認証情報に使います。

```bash
$ docker-compose run --rm iotutil owner

> iotutil@0.0.1 start /app
> node app.js "owner"

┌─────────┬────────────────────┬──────────────────────────────────────┐
│ keyword │ meshblu_auth_token │ meshblu_auth_uuid                    │
├─────────┼────────────────────┼──────────────────────────────────────┤
│ owner   │ 8ec3992b           │ ed311cd7-dc09-459b-9aaf-7d1c08186bdd │
└─────────┴────────────────────┴──────────────────────────────────────┘
```

一定間隔でレンジクエリのstartとfinishをインクリメントして、MongoDBに閾値越えのデータが保存されているか確認します。レコードタイムスタンプが`2015-07-24T10:27:01.766Z`です。テストのため`start`と`finish`でタイムスタンプをはさむようにクエリします。


```bash
$ curl -X GET  \
  "https://localhost/data/71a24381-02e2-47f3-bfa6-8174545f2261?start=2015-07-24T10:25:01.766Z&finish=2015-07-24T10:40:01.766Z&limit=1"  \
  --insecure  \
  --header "meshblu_auth_uuid: ed311cd7-dc09-459b-9aaf-7d1c08186bdd" \
  --header "meshblu_auth_token: 8ec3992b"
```

以下のようにtrigger-1がmosquitto_pubから保存したデータを取り出すことができました。

```json
{"data":[{"devices":["71a24381-02e2-47f3-bfa6-8174545f2261"],"payload":"trigger on","fromUuid":"71a24381-02e2-47f3-bfa6-8174545f2261","uuid":"71a24381-02e2-47f3-bfa6-8174545f2261","timestamp":"2015-07-24T10:27:01.766Z"}]}
```
