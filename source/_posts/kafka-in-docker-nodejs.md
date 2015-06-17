title: "Kafka in DockerにNode.jsのproducerとconsumerコンテナから接続する"
date: 2015-04-29 22:37:45
tags:
 - Kafka
 - Docker
 - DockerCompose
 - Nodejs
 - ZooKeeper
description: 前回作成したKafkaクラスタをテストするために、簡単なNode.jsのproducerとconsumer用のコンテナを作成します。追加コンテナもKafkaとZooKeeperと同じdocker-compose.ymlに含めたかったのですが、うまく動かせませんでした。producerとconsumerのコンテナは通常のdocker runコマンドで起動することにします。
---

[前回](2015/04/28/kafka-in-docker-getting-started/)作成したKafkaクラスタをテストするために、簡単なNode.jsのproducerとconsumer用のコンテナを作成します。追加コンテナもKafkaとZooKeeperと同じdocker-compose.ymlに含めたかったのですが、うまく動かせませんでした。producerとconsumerのコンテナは通常の`docker run`コマンドで起動することにします。


<!-- more -->

## kafka-node

Node.jsのKafkaクライアントはいくつかGitHubにあがっています。

* [kafka-node](https://github.com/SOHU-Co/kafka-node)
* [Prozess](https://github.com/cainus/Prozess)

今回のKafkaのバージョンは0.8.2.1です。Prozessは0.6のままので、0.8に対応しているkafka-nodを使うことにします。


## プロジェクト

最初に作成したファイルのディレクトリ構造です。適当なディレクトリを作成します。

``` bash
$ cd ~/docker_apps
$ tree
.
├── docker-compose.yml
├── kafka_consumer
│   ├── Dockerfile
│   ├── app.js
│   └── package.json
├── kafka_docker
│   ├── Dockerfile
│   ├── LICENSE
│   ├── README.md
│   ├── broker-list.sh
│   ├── docker-compose-single-broker.yml
│   ├── docker-compose.yml
│   ├── download-kafka.sh
│   ├── start-kafka-shell.sh
│   └── start-kafka.sh
└── kafka_producer
    ├── Dockerfile
    ├── app.js
    └── package.json
```

kafka_dockerは`git clone`します。

```
$ cd ~/docker_apps
$ git clone https://github.com/SOHU-Co/kafka-node.git
```

## プログラム

docker-compose.ymlは前回と変わりません。ここにproducerとconsumerのコンテナも追加したいのですが、起動の順番が制御できず動作できませんでした。
 
### docker-compose.yml

```yaml ~/docker_apps/docker-compose.yml
zookeeper:
  image: wurstmeister/zookeeper
  ports:
    - "2181"
kafka:
  build: ./kafka_docker
  ports:
    - "9092"
  links:
    - zookeeper:zk
  environment:
    KAFKA_ADVERTISED_HOST_NAME: 10.3.0.165
  volumes:
    - /var/run/docker.sock:/var/run/docker.sock
```

### Dockerfileとpackage.json

producerとconsumerのDockerfile、package.jsonは同じです。

```bash ~/docker_apps/kafka_producer/Dockerfile
FROM node:0.12-onbuild
```

package.jsonはnameとdescriptionを変更します。

```json ~/docker_apps/kafka_producer/package.json
{
  "name": "kafka-node-producer-app",
  "description": "kafka-node-producer app",
  "version": "0.0.1",
  "private": true,
  "dependencies": {
    "kafka-node": "0.2.26"
  },
  "scripts": {"start": "node app.js"}
}
```

### kafka_producer/app.js

High Levelのproducerのサンプルは[high-level-producer.js](https://github.com/SOHU-Co/kafka-node/blob/master/example/high-level-producer.js)にあります。ZooKeeperのホストとポート番号は環境変数より取得します。

```js ~/docker_apps/kafka_producer/app.js
'use strict';
var kafka = require('kafka-node'),
    HighLevelProducer = kafka.HighLevelProducer,
    Client = kafka.Client,
    host = [process.env.ZK_PORT_2181_TCP_ADDR,':',
            process.env.ZK_PORT_2181_TCP_PORT],
    client = new Client(host.join('')),
    producer = new HighLevelProducer(client),
    count = 10, rets = 0;

producer.on('ready', function () {
    setInterval(send, 1000);
});

producer.on('error', function (err) {
    console.log('error', err)
});

function send() {
    var payloads = [
        {topic: 'topic1', messages: ['hello','world']}
    ];
    producer.send(payloads, function (err, data) {
        if (err) console.log(err);
        else console.log('send %d messages', ++rets);
        if (rets === count) process.exit();
    });
}
```

#### kafka_consumer

High Levelのconsumerのサンプルも[high-level-consumer.js](https://github.com/SOHU-Co/kafka-node/blob/master/example/high-level-consumer.js)にあります。ZooKeeperの情報やオリジナルからtopic名を固定にするなど少し変更しています。


```js ~/docker_apps/kafka_consumer/app.js
'use strict';
var kafka = require('kafka-node'),
    HighLevelConsumer = kafka.HighLevelConsumer,
    Client = kafka.Client,
    host = [process.env.ZK_PORT_2181_TCP_ADDR,':',
            process.env.ZK_PORT_2181_TCP_PORT],
    client = new Client(host.join('')),
    topics = [ { topic: 'topic1' }],
    options = { autoCommit: true, fetchMaxWaitMs: 1000, fetchMaxBytes: 1024*1024 },
    consumer = new HighLevelConsumer(client, topics, options);

consumer.on('message', function (message) {
    console.log(message);
});

consumer.on('error', function (err) {
    console.log('error', err);
});
```

## Dockerコンテナの起動

Docker ComposeからKafkaとZooKeeperのコンテナを起動します。

``` bash
$ cd ~/docker_apps
$ docker-compose up
```

producerのDockerイメージをビルドしてコンテナを起動します。メッセージは10回送信します。`--links`フラグを追加してプログラムから環境変数を通してZooKeeperのIPアドレスとポート番号を取得できるようにします。ZooKeeperの名前はDocker Composeが自動的に設定しているので`docker-compose ps`コマンドから名前を確認しておきます。

``` bash
$ cd ~/docker_apps/kafka_producer
$ docker build -t kafka_producer .
$ docker run --rm --link dockerapps_zookeeper_1:zk kafka_producer
...
send 9 messages
send 10 messages
```

consumerのDockerイメージをビルドしてコンテナを起動します。producerは毎回メッセージを2つ送信しているのでconsumerには20個のメッセージが届きます。

```
$ cd ~/docker_apps/kafka_consumer
$ docker build -t kafka_consumer .
$ docker run --rm --link dockerapps_zookeeper_1:zk kafka_consumer
...
{ topic: 'topic1',
  value: 'hello',
  offset: 18,
  partition: 0,
  key: <Buffer > }
{ topic: 'topic1',
  value: 'world',
  offset: 19,
  partition: 0,
  key: <Buffer > }
```
