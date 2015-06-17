title: "Meshblu on Dockerの構築とMosquittoでテスト"
date: 2015-03-17 13:20:03
tags:
 - Octoblu
 - Meshblu
 - Nodejs
 - MQTT
 - Mosquitto
 - IoT
description: Dockerを使いMeshbluのIoTプラットフォームをローカルに構築します。GitHubのリポジトリにはDockerfileもあるのですが、開発用に`node_modules`のキャッシュと`docker restart`がやりやすいように少し構成を変更します。最後にMosquittoのコンテナを2つ使いPub/Subのテストをしてみます。
---

Dockerを使い[Meshblu](https://developer.octoblu.com/)のIoTプラットフォームをローカルに構築します。GitHubの[リポジトリ](https://github.com/octoblu/meshblu)には[Dockerfile](https://github.com/octoblu/meshblu/blob/master/Dockerfile)もあるのですが、開発用に`node_modules`のキャッシュと`docker restart`がやりやすいように少し構成を変更します。最後に[Mosquitto](http://mosquitto.org/)のコンテナを2つ使いPub/Subのテストをしてみます。

<!-- more -->

## プロジェクトの作成

[Meshblu](https://github.com/octoblu/meshblu)のリポジトリからcloneします。

``` bash
$ cd ~/docker_apps
$ git clone https://github.com/octoblu/meshblu meshblu-dev
$ cd meshblu-dev
```


## Dockerfile

最初に[リポジトリ](https://github.com/octoblu/meshblu/blob/master/Dockerfile)から修正したDockefileです。upstreamに頻繁に更新が入るのでディレクトリ構成などあまり変更しないようにしました。

``` bash ~/docker_apps/meshblu-dev/Dockerfile
FROM ubuntu:12.04

MAINTAINER Skynet https://skynet.im/ <chris+docker@skynet.im>

RUN apt-get update -y --fix-missing
RUN apt-get install -y python-software-properties
RUN apt-get install -y build-essential
RUN apt-get -y install libzmq-dev libavahi-compat-libdnssd-dev

RUN add-apt-repository ppa:chris-lea/node.js

RUN apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv 7F0CEB10
RUN echo "deb http://downloads-distro.mongodb.org/repo/ubuntu-upstart dist 10gen" | tee -a /etc/apt/sources.list.d/10gen.list
RUN apt-get update -y --fix-missing
RUN apt-get -y install redis-server apt-utils supervisor nodejs
RUN apt-get -y install -o apt::architecture=amd64 mongodb-10gen

RUN sed -i 's/daemonize yes/daemonize no/g' /etc/redis/redis.conf

#ADD . /var/www
#RUN cd /var/www && npm install

COPY ./package.json /var/www/
RUN mkdir -p /dist/node_modules && \
  ln -s /dist/node_modules /var/www/node_modules && \
  cd /var/www && npm install

#ADD ./docker/config.js.docker /var/www/config.js
#ADD ./docker/supervisor.conf /etc/supervisor/conf.d/supervisor.conf
RUN mkdir /var/log/skynet

EXPOSE 3000 5683 1883

CMD ["/usr/bin/supervisord", "-n", "-c", "/etc/supervisor/supervisord.conf"]
VOLUME /etc/supervisor/conf.d
COPY . /var/www
```

### 不足パッケージの追加

`docker build`中にエラーが出たので、zmq.hとdns_sd.hをインストールします。

``` bash ~/docker_apps/meshblu-dev/Dockerfile
RUN apt-get -y install libzmq-dev libavahi-compat-libdnssd-dev
```

### node_modulesのシムリンク

以下の方針でDockerfileを修正します。

* `node_modules`のディレクトリを`/dist/node_modules`に作成して、アプリケーションディレクトリにはシムリンクを作る
* Dockerfileの最後にCOPYコマンドでキャッシュして、コード変更後にnpmインストールが起きないようにする

``` bash ~/docker_apps/meshblu-dev/Dockerfile
#ADD . /var/www
#RUN cd /var/www && npm install

COPY ./package.json /var/www/
RUN mkdir -p /dist/node_modules && \
  ln -s /dist/node_modules /var/www/node_modules && \
  cd /var/www && npm install
```

* ./dockerの設定ファイルはCOPYしない
* カレントディレクトリのconfig.jsを修正してビルドする

``` bash ~/docker_apps/meshblu-dev/Dockerfile
#ADD ./docker/config.js.docker /var/www/config.js
#ADD ./docker/supervisor.conf /etc/supervisor/conf.d/supervisor.conf
```

* supervisor.confのボリューム作成と、最後にカレントディレクトリをイメージにキャッシュする

``` bash ~/docker_apps/meshblu-dev/Dockerfile
CMD ["/usr/bin/supervisord", "-n", "-c", "/etc/supervisor/supervisord.conf"]
VOLUME /etc/supervisor/conf.d
COPY . /var/www
```

### MQTTのポート追加

exposeにMQTTの1883ポートを追加します。

``` bash ~/docker_apps/meshblu-dev/Dockerfile
EXPOSE 3000 5683 1883
```

## Dockerホストのシムリンク

前回[node_modulesのシムリンクとボリュームを使って効率化](/2015/03/13/node-modules-symlinks-in-docker/)したように、`node_modules`コンテナ内でのみ有効なシムリンクをDockerホスト上の作業ディレクトリ内に作成します。

``` bash
$ ln -s /dist/node_modules ./node_modules
```

## supervisor.conf

Supervisorの設定ファイルは以下のように修正しました。

``` bash ~/docker_apps/meshblu-dev/docker/supervisor.conf
[program:redis]
command=/usr/bin/redis-server /etc/redis/redis.conf
numprocs=1
autostart=true
autorestart=true

#[program:mongodb]
#command=/usr/bin/mongod --config /etc/mongodb.conf
#numprocs=1
#autostart=true
#autorestart=true

[program:node]
#command=/usr/bin/node /var/www/server.js --http --coap
command=/usr/bin/node /var/www/server.js --http --coap --mqtt
numprocs=1
directory=/var/www/
stdout_logfile=/dev/fd/1
stdout_logfile_maxbytes=0
redirect_stderr=true
autostart=true
autorestart=true
```

### MQTT Serverを起動する

server.js起動時に`--mqtt`フラグを追加してMQTTサーバーを起動します。

``` bash ~/docker_apps/meshblu-dev/docker/supervisor.conf
[program:node]
#command=/usr/bin/node /var/www/server.js --http --coap
command=/usr/bin/node /var/www/server.js --http  --coap --mqtt
```

### MongoDBは起動しない

今回はRedisだけ使うのでMongoDBはSupervisorから起動させません。

``` bash ~/docker_apps/meshblu-dev/docker/supervisor.conf
#[program:mongodb]
#command=/usr/bin/mongod --config /etc/mongodb.conf
#numprocs=1
#autostart=true
#autorestart=true
```

## config.js

今回はリポジトリのディレクトリ構成を残す方針です。アプリケーション用にディレクトリを作成しないので、カレントディレクトリに`node_modules`も配置します。カレントディレクトリごとコンテナの配布ディレクトリにボリュームでアタッチします。`docker/config.js.docker`でなく`./config.js`を修正します。

``` js ~/docker_apps/meshblu-dev/config.js
...
module.exports = {
  /*
  mongo: {
    databaseUrl: process.env.MONGODB_URI
  },
  */
  port: parseInt(process.env.PORT) || 80,
  /*
  tls: {
    sslPort: parseInt(process.env.SSL_PORT) || 443,
    cert: process.env.SSL_CERT,
    key: process.env.SSL_KEY
  },
  */
...
  mqtt: {
    //databaseUrl: process.env.MQTT_DATABASE_URI,
    port: parseInt(process.env.MQTT_PORT),
    skynetPass: process.env.MQTT_PASSWORD
  },
```

### MongoDBは使わない

config.mqtt.databaseUrlはコメントアウトするとMondoDBは使われなくなります。Node.jsのコードを読むとMQTTのバックエンドはRedisが設定されていれば優先して使用されているようです。

``` js ~/docker_apps/meshblu-dev/config.js
module.exports = {
  /*
  mongo: {
    databaseUrl: "mongodb://localhost:27017/skynet",
  },
  */
```

### TLS/SSLは保留

証明書をまだ用意していないのでTLS/SSLの設定はコメントアウトします。

``` js ~/docker_apps/meshblu-dev/config.js
  /*
  tls: {
    sslPort: parseInt(process.env.SSL_PORT) || 443,
    cert: process.env.SSL_CERT,
    key: process.env.SSL_KEY
  },
  */
```

### MQTT Server用にskynetPassなど設定

システム内部のメッセージングにskynetユーザーを使っています。そのためMQTTサーバー用にskynetPassの指定は必須です。起動メッセージから送信できなくなりサーバーが起動しません。ランダムパスワードを作成して設定します。

``` ruby
irb(main):001:0> o = [('a'..'z'), ('A'..'Z')].map { |i| i.to_a }.flatten
=> ["a", "b", "c", "d", "e", "f", "g", "h", "i", "j", "k", "l", "m", "n", "o", "p", "q", "r", "s", "t", "u", "v", "w", "x", "y", "z", "A", "B", "C", "D", "E", "F", "G", "H", "I", "J", "K", "L", "M", "N", "O", "P", "Q", "R", "S", "T", "U", "V", "W", "X", "Y", "Z"]
irb(main):002:0> (0...44).map { o[rand(o.length)] }.join
=> "ZFiLQKSmFmaDgFVrZyTCVwsBUCUpWhdHhjVBoGxNnHaa"
```

## Dockerイメージのビルドと起動

Dockerイメージをビルドします。

``` bash
$ docker build -t meshblu-dev .
```

環境変数にポート番号やskynetPassを指定します。カレントディレクトリやSupervisorの設定ファイルをボリュームでアタッチして起動します。

``` bash
$ docker run -d --name meshblu-dev \
  -p 3000:3000 \
  -p 5683:5683 \
  -p 1883:1883 \
  -e PORT=3000 \
  -e MQTT_PORT=1883 \
  -e MQTT_PASSWORD=ZFiLQKSmFmaDgFVrZyTCVwsBUCUpWhdHhjVBoGxNnHaa \
  -v $PWD:/var/www \
  -v $PWD/docker:/etc/supervisor/conf.d \
  meshblu-dev
```

ログを確認します。HTTP(3000), CoAP(5683), MQTT(1883)のサーバーが起動しました。

``` bash
$ docker logs -f meshblu-dev
...
MM    MM              hh      bb      lll
MMM  MMM   eee   sss  hh      bb      lll uu   uu
MM MM MM ee   e s     hhhhhh  bbbbbb  lll uu   uu
MM    MM eeeee   sss  hh   hh bb   bb lll uu   uu
MM    MM  eeeee     s hh   hh bbbbbb  lll  uuuu u
                 sss
Meshblu (formerly skynet.im) development environment loaded...

Starting CoAP... done.
Starting HTTP/HTTPS... done.
...
Starting MQTT... done.
CoAP listening at coap://localhost:5683
HTTP listening at http://0.0.0.0:3000
...
MQTT listening at mqtt://0.0.0.0:1883
...
```

Node.jsファイルなどを修正したら、docker restartで反映ができます

``` bash
$ docker restart meshblu-dev
```

## デバイス/ノードの登録と簡単なPub/Sub

REST APIからMeshbluのステータスを確認します。

``` bash
$ curl http://localhost:3000/status
{"meshblu":"online"}
```

コネクテッドデバイスを用意する前に、ノードとしてDockerコンテナを使いテストします。[Mosquitto](http://mosquitto.org/)のPublishとSubscribe用の2つのコンテナを起動します。

### MQTTノードの登録

REST APIを使ってノードの登録をします。ボディには任意のkey/valueを登録できます。まず`mqtt-pub`コンテナの登録をします。戻り値のJSONから`uuid`と`token`を取得できます。今後このデバイス/ノードとメッセージをやりとりする場合にはこの`uuid`と`token`が必要になります。

``` bash
$ curl -X POST -d "name=mqtt-pub&type=container" "http://localhost:3000/devices"
{"uuid":"1f3bcae0-cc5b-11e4-80ac-dbe1b8e07fba","online":false,"timestamp":"2015-03-17T04:07:26.621Z","name":"mqtt-pub","type":"container","ipAddress":"172.17.42.1","token":"1fc767197e275a3cb79befbf38632e5c7ef8fbec"}
```

`mqtt-sub`コンテナも同様に登録します。

``` bash
$ curl -X POST -d "name=mqtt-sub&type=container" "http://localhost:3000/devices"
{"uuid":"38cc2270-cc5b-11e4-80ac-dbe1b8e07fba","online":false,"timestamp":"2015-03-17T04:08:09.496Z","name":"mqtt-sub","type":"container","ipAddress":"172.17.42.1","token":"86efacbbeb28dd56676df710cf4c1e7d2f05f762"}
```

登録したデバイスを確認します。HTTPヘッダには`uuid/token`の認証情報が必要です。今回は`mqtt-sub`のuuidを認証に使いましたが、`mqtt-pub`のデバイス情報も取得できています。uuidにはオーナーを指定すると所有しているデバイスは操作できるようになるのですが、この辺りは調査中です。プライベートなブローカーなので今のところすべてのデバイスが見えても問題ありません。

``` bash
$ curl "http://localhost:3000/devices" \
  --header "meshblu_auth_uuid: 38cc2270-cc5b-11e4-80ac-dbe1b8e07fba" \
  --header "meshblu_auth_token: 86efacbbeb28dd56676df710cf4c1e7d2f05f762"
{"devices":[{"uuid":"38cc2270-cc5b-11e4-80ac-dbe1b8e07fba","online":false,"timestamp":"2015-03-17T04:08:09.496Z","name":"mqtt-sub","type":"container","ipAddress":"172.17.42.1"},{"uuid":"1f3bcae0-cc5b-11e4-80ac-dbe1b8e07fba","online":false,"timestamp":"2015-03-17T04:07:26.621Z","name":"mqtt-pub","type":"container","ipAddress":"172.17.42.1"}]}
```

### Mosquittoコンテナを起動

Mosquittoのコンテナを2つ起動します。Dockerイメージは[ansi/mosquitto](https://registry.hub.docker.com/u/ansi/mosquitto/)をpullして使います。1つ目は`mqtt-pub`コンテナです。

``` bash
$ docker pull ansi/mosquitto
$ docker run --rm --name mqtt-pub -it ansi/mosquitto /bin/bash
root@b0468fe6e67e:/usr/local/src/mosquitto-1.3#
```

別のシェルを開き、もう一つ`mqtt-sub`コンテナを起動します。

``` bash
$ docker run --rm --name mqtt-sub -it ansi/mosquitto /bin/bash
root@12b53215b59a:/usr/local/src/mosquitto-1.3#
```

このコンテナは`/etc/ld.so.cache`が更新されていないようなので`mosquitto_sub`コマンドを実行するとエラーになります。

``` bash
mosquitto_sub: error while loading shared libraries: libmosquitto.so.1: cannot open shared object file: No such file or directory
```

ldconfigを実行して共有ライブラリを変更します。

``` bash
$ ldconfig
```

### Subscribe (topicはデバイスのuuid)

Publish側コンテナ(`mqtt-sub`)で`mosquitto_sub`コマンドを実行します。任意のtopicをsubscribeするのではなく、デバイスのuuidのtopicを指定することに注意します。

* host: MeshbluのポートをマップしているDockerホスト
* port: MQTT Serverのポート
* topic: subscribeするデバイスのuuid
* username: subscribeするデバイスのuuid
* password: subscribeするデバイスのtoken

``` bash
$ MQTT_SUB_UUID=38cc2270-cc5b-11e4-80ac-dbe1b8e07fba
$ MQTT_SUB_PASS=86efacbbeb28dd56676df710cf4c1e7d2f05f762
$ mosquitto_sub -h 10.3.0.230 -p 1883 \
  -t $MQTT_SUB_UUID \
  -u $MQTT_SUB_UUID \
  -P $MQTT_SUB_PASS \
  -d
Client mosqsub/17-12b53215b59a sending CONNECT
Client mosqsub/17-12b53215b59a received CONNACK
Client mosqsub/17-12b53215b59a sending SUBSCRIBE (Mid: 1, Topic: 38cc2270-cc5b-11e4-80ac-dbe1b8e07fba, QoS: 0)
Client mosqsub/17-12b53215b59a received SUBACK
Subscribed (mid: 1): 0
Client mosqsub/17-12b53215b59a sending PINGREQ
Client mosqsub/17-12b53215b59a received PINGRESP
```

### Publish (topicはmessage)


Publish側コンテナ(`mqtt-pub`)で`mosquitto_pub`コマンドを実行します。メッセージを送信する場合のtopicは`message`で固定になっています。

* host: MeshbluのポートをマップしているDockerホスト
* port: MQTT Serverのポート
* topic: `message`で固定
* username: publishするデバイスのuuid
* password: publishするデバイスのtoken
* message: JSON形式のメッセージ、{"devices": ["xxx","yyy"], "payload": {"aaa":"bbb"}}

publishするメッセージはJSON形式でフォーマットが決まっています。`devices`キーには送信先デバイスのuuidを指定します。Arrayで複数指定も可能です。`payload`もキーが固定です。これ以外ではメッセージが送信ができないので注意が必要です。

``` bash
$ MQTT_SUB_UUID=38cc2270-cc5b-11e4-80ac-dbe1b8e07fba
$ MQTT_PUB_UUID=1f3bcae0-cc5b-11e4-80ac-dbe1b8e07fba
$ MQTT_PUB_PASS=1fc767197e275a3cb79befbf38632e5c7ef8fbec
$ mosquitto_pub -h 10.3.0.230 -p 1883 \
  -t message \
  -m '{"devices": "38cc2270-cc5b-11e4-80ac-dbe1b8e07fba", "payload": {"red":"on"}}'\
  -u 1f3bcae0-cc5b-11e4-80ac-dbe1b8e07fba \
  -P 1fc767197e275a3cb79befbf38632e5c7ef8fbec \
  -d
Client mosqpub/19-b0468fe6e67e sending CONNECT
Client mosqpub/19-b0468fe6e67e received CONNACK
Client mosqpub/19-b0468fe6e67e sending PUBLISH (d0, q0, r0, m1, 'message', ... (76 bytes))
Client mosqpub/19-b0468fe6e67e sending DISCONNECT
```

### Subscribe側コンテナでメッセージを受信

Subscribe側コンテナ(`mqtt-sub`)にメッセージが届き標準出力します。デバイスのuuidをtopicにしてsubscribeしているのですが、`message`のtopicにpublishしています。ちょっとわかりずらいですがマルチプロトコルに対応するためこういった仕様になっているようです。

``` bash
...
Client mosqsub/17-12b53215b59a received PUBLISH (d0, q0, r0, m0, '38cc2270-cc5b-11e4-80ac-dbe1b8e07fba', ... (150 bytes))
{"topic":"message","data":{"devices":"38cc2270-cc5b-11e4-80ac-dbe1b8e07fba","payload":{"red":"on"},"fromUuid":"1f3bcae0-cc5b-11e4-80ac-dbe1b8e07fba"}}
```
