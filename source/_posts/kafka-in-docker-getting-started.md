title: "Kafka in Dockerで分散メッセージングシステムを構築する"
date: 2015-04-28 11:24:13
tags:
 - Kafka
 - Docker
 - DockerCompose
 - Arduino
 - ZooKeeper
description: Apach KafkaはLinkedInで開発された分散メッセージングシステムです。ArduinoからMQTTブローカーにpublishしたセンシングデータをKafkaでconsumeしてからRiemann、Spark Streaming、Stormなどのリアルタイムストリーミング処理が目的です。まずはDockerでKafkaクラスタを構築します。
---

[Apach Kafka](http://kafka.apache.org/)は[LinkedIn](http://data.linkedin.com/opensource/kafka)で開発された分散メッセージングシステムです。ArduinoからMQTTブローカーにpublishしたセンシングデータをKafkaでconsumeしてから[Riemann](http://riemann.io/index.html)、[Spark Streaming](https://spark.apache.org/streaming/)、[Storm](https://storm.apache.org/)などのリアルタイムストリーミング処理が目的です。

<!-- more -->

## Docker Composeのインストール

KafkaのDockerイメージは[wurstmeister/kafka-docker](https://registry.hub.docker.com/u/wurstmeister/kafka/)を使います。Kafkaクラスタの構成管理にZookeeperが必要になります。このイメージはDocker Composeを使っているのでちょうど良い勉強になります。事前にインストールしておきます。

``` bash
$ curl -L https://github.com/docker/compose/releases/download/1.2.0/docker-compose-`uname -s`-`uname -m` > /usr/local/bin/docker-compose; chmod +x /usr/local/bin/docker-compose
```

## プロジェクトの作成

適当なディレクトリを作成して、[リポジトリ](https://github.com/wurstmeister/kafka-docker)から`git clone`します。

``` bash
$ cd ~/docker_apps
$ git clone https://github.com/wurstmeister/kafka-docker.git
$ cd kafka-docker
```

## クラスタを起動する

[Kafka Docker](http://wurstmeister.github.io/kafka-docker/)の手順を読みながらクラスタの構築と簡単なテストまで行います。


### docker-compose.yml

[リポジトリ](https://github.com/wurstmeister/kafka-docker)にはクラスタ用と1台構成用のdocker-compose.ymlが用意されています。今回はブローカーを2台起動したいのでクラスタ用のdocker-compose.ymlを使います。YAMLの`KAFKA_ADVERTISED_HOST_NAME`の値をDockerホストのIPアドレスに書き換えて実行します。


```yaml ~/docker_apps/kafka-docker/docker-compose.yml
zookeeper:
  image: wurstmeister/zookeeper
  ports:
    - "2181"
kafka:
  build: .
  ports:
    - "9092"
  links:
    - zookeeper:zk
  environment:
    KAFKA_ADVERTISED_HOST_NAME: 10.3.0.165
  volumes:
    - /var/run/docker.sock:/var/run/docker.sock
```

最初に`docker-compose up`するとkafkaのDockerイメージのビルドが始まります。必要なコンテナがすべて起動するまでしばらく待ちます。

``` bash
$ docker-compose up
```

クラスタがすべて起動したら別シェルからKafkaブローカーを2台に変更します。

``` bash
$ docker-compose scale kafka=2
Creating kafkadocker_kafka_2...
Starting kafkadocker_kafka_2...
```

`ps`でコンテナの起動状況を確認します。

``` bash
$ docker-compose ps
         Name                     Command                    State                     Ports
-----------------------------------------------------------------------------------------------------
kafkadocker_kafka_2       /bin/sh -c start-         Up                        0.0.0.0:32789->9092/tcp
                          kafka.sh
kafkadocker_kafka_3       /bin/sh -c start-         Up                        0.0.0.0:32790->9092/tcp
                          kafka.sh
kafkadocker_zookeeper_1   /bin/sh -c                Up                        0.0.0.0:32788->2181/tcp
                          /usr/sbin/sshd  ...                                 , 22/tcp, 2888/tcp,
                                                                              3888/tcp
```

## Kafka Shell

Kafka Shellを起動する書式は以下です。

``` bash
$ start-kafka-shell.sh <DOCKER_HOST_IP> <ZK_HOST:ZK_PORT>
```

`ZK_HOST:PORT`の値は`docker-compose ps`で確認した値を使います。

``` bash
$ cd ~/docker_apps/kafka-docker/
$ ./start-kafka-shell.sh 10.3.0.165 10.3.0.165:32788
```

`start-kafka-shell.sh`では使い捨てのDockerコンテナを起動してbashを実行します。このコンテナを使いKafkaのproducerとconsumerのプロセスを起動します。

Kafkaコンテナの環境変数を確認しておきます。

``` bash
$ echo $KAFKA_HOME
/opt/kafka_2.10-0.8.2.0
$ echo $ZK
10.3.0.165:32788
```

### topicの作成

Kafka Shellを使いtopicを作成します。Kafkaのtopicはメッセージのカテゴリになります。このtopicはパーティションは4つ、メッセージのレプリカは2個の設定です。

``` bash
$ $KAFKA_HOME/bin/kafka-topics.sh --create --topic topic \
--partitions 4 --zookeeper $ZK --replication-factor 2
Created topic "topic".
```

作成したtopicを確認します。

```
$ $KAFKA_HOME/bin/kafka-topics.sh --describe --topic topic --zookeeper $ZK
Topic:topic     PartitionCount:4        ReplicationFactor:2     Configs:
        Topic: topic    Partition: 0    Leader: 1       Replicas: 1,2   Isr: 1,2
        Topic: topic    Partition: 1    Leader: 2       Replicas: 2,1   Isr: 2,1
        Topic: topic    Partition: 2    Leader: 1       Replicas: 1,2   Isr: 1,2
        Topic: topic    Partition: 3    Leader: 2       Replicas: 2,1   Isr: 2,1
```

### producerの送信 

Kafka Shellの一つからproducerを実行します。コンソールは文字列の入力待ちになります。

``` bash
$ $KAFKA_HOME/bin/kafka-console-producer.sh --topic=topic \
--broker-list=`broker-list.sh`
```

### consumerの受信

Kafka Shellを起動してconsumerを実行します。

``` bash
$ ./start-kafka-shell.sh 10.3.0.165 10.3.0.165:32788
$ $KAFKA_HOME/bin/kafka-console-consumer.sh --topic=topic --zookeeper=$ZK
```

入力待ちのproducerのコンソールに文字列をタイプすると、consumerのコンソールに表示されます。

