title: "MeshbluのdebugとMQTT-HTTP BridgeのためのRedis"
date: 2015-03-25 23:37:34
tags:
 - MQTT
 - Meshblu
 - Redis
 - Supervisor
 - Mosquitto
description: 何回かMeshbluのコンテナを再作成して作業しているとMQTT-HTTP Bridgeが突然動かなくなりました。はじめはNode.jsのコードのデバッグに適当なところでconsole.logしていたのですが、Meshbluではdebugパッケージを使っているのでこれを活用してみます。結局MQTT-HTTP Bridgeが動かなくなったのはRedisと連携する設定が抜けていたためでした。
---

何回かMeshbluのコンテナを再作成して作業しているとMQTT-HTTP Bridgeが突然動かなくなりました。はじめはNode.jsのコードのデバッグに適当なところでconsole.logしていたのですが、Meshbluではdebugパッケージを使っているのでこれを活用してみます。結局MQTT-HTTP Bridgeが動かなくなったのはRedisと連携する設定が抜けていたためでした。

<!-- more -->

## debugパッケージ

Meshbluの一部のコードでは[debug](https://github.com/visionmedia/debug)パッケージをrequireしています。名前を引数にしてdebug関数を作成します。

``` js worker.js
var debug = require('debug')('worker');

setInterval(function(){
  debug('doing some work');
}, 1000);
```

デバッグを有効にする場合はDEBUG環境変数にdebug関数で引数にした名前を設定します。

``` bash
DEBUG=worker node example/app
```

複数設定する場合はカンマ区切りします。

``` bash
DEBUG=worker,http node example/app
```

### 使い方

Mesubluの場合debug関数は以下のように作成しています。

```js ~/docker_apps/meshblu/lib/sendMessage.js
var debug = require('debug')('meshblu:sendMessage');
```

Node.jsのプログラムはSupervisorから実行しているので`supervisor.conf`に環境変数を追加します。

``` bash ~/docker_apps/meshblu/docker/supervisor.conf
[program:node]
environment=DEBUG="meshblu:sendMessage"
command=/usr/bin/node /var/www/server.js --http --coap --mqtt
```

デバッグしたいところにdebug関数を使います。

```js ~/docker_apps/meshblu/lib/sendMessage.js
            debug('Sending message', emitMsg);
            debug('protocol',check.protocol);
            if(check.protocol === 'mqtt'){
              mqttEmitter(check.uuid, wrapMqttMessage(topic, emitMsg), {qos: data.qos || DEFAULT_QO\
S});
            }
            else{
              socketEmitter(check.uuid, topic, emitMsg);
            }
```

`docker logs`にデバッグが表示されるようになります。

``` bash
$ docker logs -f meshblu
Wed, 25 Mar 2015 14:30:53 GMT meshblu:sendMessage Sending message { devices: [ 'my-iot' ],
  payload: { yello: 'on' },
  fromUuid: 'temp-sens' }
Wed, 25 Mar 2015 14:30:53 GMT meshblu:sendMessage protocol undefined
```

## Redisの有効化

デバッグしていくと`mqttServer.js`のsocketEmitter関数でemitがされていないようです。この関数はprotocolがhttpに設定されているデバイスに対してMQTTからHTTPにブリッジするときに実行されます。ブリッジされなかったのはRedisの設定が`config.js`から読み取れないためｄした。

```js ~/docker_apps/meshblu/lib/mqttServer.js
  if(config.redis && config.redis.host){
    var redis = require('./redis');
    io = require('socket.io-emitter')(redis.client);
  }
...
  function socketEmitter(uuid, topic, data){
    if(io){
      io.in(uuid).emit(topic, data);
    }
  }
```

redisの設定値は環境変数から読み込んでいます。

```js ~/docker_apps/meshblu/config.js
  redis: {
    host: process.env.REDIS_HOST,
    port: parseInt(process.env.REDIS_PORT),
    password: process.env.REDIS_PASSWORD
  },
```

`docker run`の環境変数にRedisの設定を追加して起動します。

``` bash
 docker run -d --name meshblu \
  -p 3000:3000 \
  -p 4443:4443 \
  -p 5683:5683 \
  -p 1883:1883 \
  -e PORT=3000 \
  -e MQTT_PORT=1883 \
  -e MQTT_PASSWORD=skynetpass \
  -e MONGODB_URI=mongodb://localhost:27017/skynet \
  -e REDIS_HOST=localhost \
  -e REDIS_PASSWORD=localhost \
  -e SSL_PORT=4443 \
  -e SSL_CERT=/opt/meshblu/certs/server.crt \
  -e SSL_KEY=/opt/meshblu/certs/server.key \
  -v $PWD:/var/www \
  -v $PWD/docker:/etc/supervisor/conf.d \
  -v $HOME/docker_apps/certs/meshblu:/opt/meshblu/certs \
  meshblu
```

### pub/sub

最初にpublishとsubscribeのデバイスのUUID/TOKENを`~/.bashrc`へ環境変数に設定します。

``` bash ~/.bashrc
export TEMP_SENS_UUID=temp-sens
export TEMP_SENS_TOKEN=123
export MY_IOT_UUID=my-iot
export MY_IOT_TOKEN=456
```

最初にcurlでsubscribeします。

``` bash
$ curl -X GET \
  "http://localhost:3000/subscribe" \
  --insecure \
  --header "meshblu_auth_uuid: $MY_IOT_UUID" \
  --header "meshblu_auth_token: $MY_IOT_TOKEN"
```

次に新しいシェルを開きMosquittoからpublishします。

``` bash
$ mosquitto_pub \
  -h localhost  \
  -p 1883 \
  -t message \
  -m '{"devices": ["'"$MY_IOT_UUID"'"], "payload": {"yello":"on"}}' \
  -u $TEMP_SENS_UUID \
  -P $TEMP_SENS_TOKEN \
  -d
Client mosqpub/39356-minion1.c sending CONNECT
Client mosqpub/39356-minion1.c received CONNACK
Client mosqpub/39356-minion1.c sending PUBLISH (d0, q0, r0, m1, 'message', ... (50 bytes))
Client mosqpub/39356-minion1.c sending DISCONNECT
```

curlでsubscribeしているシェルにメッセージが標準出力されました。

``` bash
{"devices":["my-iot"],"payload":{"yello":"on"},"fromUuid":"temp-sens"},
```
