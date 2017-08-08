title: "KafkaからTreasure DataにブリッジするDocker Compose"
date: 2017-08-02 12:24:03
categories:
 - IoT
tags:
 - TreasureData
 - Kafka
description: td-agentコンテナとKafka Consumerコンテナを使いKafkaからTreasure DataへブリッジするDocker Composeサービスを起動します。別のポストではPySpark Streamingのウィンドウ集計した結果をKafkaのトピックに出力するコードを書きました。このストリーム処理はデータパイプラインの前処理やエンリッチメントに相当します。後続にビッグデータのバッチ処理を想定してTreasure Dataに保存します。
---

　td-agentコンテナとKafka Consumerコンテナを使いKafkaから[Treasure Data](https://www.treasuredata.co.jp/)へブリッジするDocker Composeサービスを起動します。[別のポスト](https://masato.github.io/2017/08/02/sensortag-kafka-python-spark-streaming-6/)ではPySpark Streamingのウィンドウ集計した結果をKafkaのトピックに出力するコードを書きました。このストリーム処理はデータパイプラインの前処理やエンリッチメントに相当します。後続にビッグデータのバッチ処理を想定してTreasure Dataに保存します。


<!-- more -->


## Docker Compose

　最初に今回作成するプロジェクトのディレクトリ構成です。

```bash
$ tree -a
.
├── docker-compose.yml
├── .env
├── .gitignore
├── kafka-bridge
│   ├── Dockerfile
│   ├── fluentd-consumer.properties
│   └── log4j.properties
└── td-agent2
    ├── Dockerfile
    └── td-agent.conf

2 directories, 9 files

```

### docker-compose.yml

　td-agentとKafka ConsumerサービスはそれぞれDockefileを書いてビルドします。Kafkaは[landoop/fast-data-dev](https://hub.docker.com/r/landoop/fast-data-dev/)を利用します。[Confluent Open Source](https://www.confluent.io/product/confluent-open-source/)を同梱しているためKafkaとZooKeeperも起動します。


```yaml docker-compose.yml
version: '2'
services:
  kafka-stack:
    image: landoop/fast-data-dev
    environment:
      - FORWARDLOGS=0
      - RUNTESTS=0
      - ADV_HOST=<仮想マシンのパブリックIPアドレス>
    ports:
      - 3030:3030
      - 9092:9092
      - 2181:2181
      - 8081:8081
  td-agent2:
    build: ./td-agent2
    env_file:
      - ./.env
    ports:
      - 24224:24224
  kafka-bridge:
    build: ./kafka-bridge
    depends_on:
      - td-agent
```

### .env

　Treasure Dataの接続情報は環境変数ファイルの`.env`に記述しDocker Composeから読み込みます。

```td-agent2/.env
TD_API_KEY=<YOUR API KEY>
TD_ENDPOINT=<TD ENDPOINT>
```

## td-agent2

　td-agentのDockerイメージを作成します。

### Dockerfile

　[Overview of Server-Side Agent (td-agent)](https://docs.treasuredata.com/articles/td-agent)のインストール手順に従います。[install-ubuntu-xenial-td-agent2.sh](https://toolbelt.treasuredata.com/sh/install-ubuntu-xenial-td-agent2.sh)の中ではsudoも必要です。

```td-agent2/Dockerfile
FROM ubuntu:xenial

RUN apt-get update && apt-get install sudo curl -y
RUN curl -L https://toolbelt.treasuredata.com/sh/install-ubuntu-xenial-td-agent2.sh | sh
RUN apt-get clean && rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*

ADD td-agent.conf /etc/td-agent/
EXPOSE 24224
CMD ["/usr/sbin/td-agent"]
```

### td-agent.conf

　td-agent.confは環境変数を[参照](https://docs.fluentd.org/v0.12/articles/faq#how-can-i-use-environment-variables-to-configure-parameters-dynamically)することができます。Treasure Dataへの接続情報を`.env`ファイルから取得します。

```td-agent2/td-agent.conf
<match td.*.*>
  @type tdlog
  endpoint "#{ENV['TD_ENDPOINT']}"
  apikey "#{ENV['TD_API_KEY']}"
  auto_create_table
  buffer_type file
  buffer_path /var/log/td-agent/buffer/td
  use_ssl true
  num_threads 8
</match>

<source>
  @type forward
</source>
```

## kafka-fluentd-consumer

　KafkaからTreasure Dataへのブリッジには[kafka-fluentd-consumer](https://github.com/treasure-data/kafka-fluentd-consumer)のJarを利用します。

### Dockerfile

 コンパイル済の[kafka-fluentd-consumer-0.3.1-all.jar](https://github.com/treasure-data/kafka-fluentd-consumer/releases/download/v0.3.1/kafka-fluentd-consumer-0.3.1-all.jar)をダウンロードします。

```kafka-bridge/Dockerfile
FROM java:8-jre
ARG KAFKA_FLUENTD_CONSUMER_VERSION=0.3.1

WORKDIR /app

RUN wget -q -O kafka-fluentd-consumer-all.jar https://github.com/treasure-data/kafka-fluentd-consumer/releases/download/v$KAFKA_FLUENTD_CONSUMER_VERSION/kafka-fluentd-consumer-$KAFKA_FLUENTD_CONSUMER_VERSION-all.jar

ADD log4j.properties .
ADD fluentd-consumer.properties .

CMD ["java", "-Dlog4j.configuration=file:///app/log4j.properties", "-jar", "kafka-fluentd-consumer-all.jar", "fluentd-consumer.properties"]
```

### fluentd-consumer.properties

　[デフォルト](https://github.com/treasure-data/kafka-fluentd-consumer/blob/master/config/fluentd-consumer.properties)の設定から以下を変更します。`fluentd.connect`と`zookeeper.connect`はdocker-compose.ymlを使う場合はそれぞれサービス名を指定します。

* fluentd.connect=<td-agentのホスト名>:24224
* fluentd.tag.prefix=td.<データベース名>.
* fluentd.consumer.topics=<トピック名>
* zookeeper.connect=<ZooKeeperのホスト名>:2181
* group.id=<コンシューマグループ名>

```kafka-bridge/fluentd-consumer.properties
# Fluentd instance destinations.
fluentd.connect=td-agent2:24224

# Dynamic event tag with topic name. 
fluentd.tag.prefix=td.sensortag_dev.

# Consumed topics. 
fluentd.consumer.topics=sensortag-sink

# The number of threads per consumer streams
fluentd.consumer.threads=1

# The path for backup un-flushed events during shutdown.
fluentd.consumer.backup.dir=/tmp/fluentd-consumer-backup/

# Kafka Consumer related parameters
zookeeper.connect=kafka-stack:2181
group.id=my-sensortag-sink-group
zookeeper.session.timeout.ms=400
zookeeper.sync.time.ms=200
auto.commit.interval.ms=1000
```

### log4j.properties

　[log4j.properties](https://github.com/treasure-data/kafka-fluentd-consumer/blob/master/config/log4j.properties)はデフォルトのまま使います。

```kafka-bridge/log4j.properties
# log4j logging configuration.
# This is based on Pinterest's secor

# root logger.
log4j.rootLogger=DEBUG, ROLLINGFILE

log4j.appender.ROLLINGFILE = org.apache.log4j.RollingFileAppender
log4j.appender.ROLLINGFILE.Threshold=INFO
log4j.appender.ROLLINGFILE.File=/tmp/fluentd-consumer.log
# keep log files up to 1G
log4j.appender.ROLLINGFILE.MaxFileSize=20MB
log4j.appender.ROLLINGFILE.MaxBackupIndex=50
log4j.appender.ROLLINGFILE.layout=org.apache.log4j.PatternLayout
log4j.appender.ROLLINGFILE.layout.ConversionPattern=%d{ISO8601} [%t] (%C:%L) %-5p %m%n
```

## 動作確認

　Docker Composeのサービスを起動します。

```bash
$ docker-compose up -d
```

　td-agentのバージョンを確認します。

```bash
$ docker-compose exec td-agent2 td-agent --version
td-agent 0.12.35
```

　Spark Streamingを使ったウィンドウ集計の[サンプル](https://masato.github.io/2017/08/02/sensortag-kafka-python-spark-streaming-6/)のように`fluentd.consumer.topics`に指定したtopicへJSONフォーマットでデータを送信します。

　テストとして`kafka-console-producer`から直接JSONを送信してみます。

```bash
$ docker-compose exec kafka-stack kafka-console-producer \
    --broker-list localhost:9092 \
    --topic sensortag-sink
```

　コマンドを実行後の待機状態でJSON文字列を入力します。

```json
{"bid": "B0:B4:48:BD:DA:03", "time": 1501654353, "humidity": 27.152099609375, "objecttemp": 21.6875, "ambient": 27.09375, "rh": 78.4423828125}
```

　td-agentはファイルバッファを作成してデフォルトでは5分間隔でTreasure Dataへデータがアップロードします。


```bash
$ docker-compose exec td-agent2 ls /var/log/td-agent/buffer
td.sensortag_dev.sensortag_sink.b555bf24951c65554.log
```

![treasuredata.png](/2017/08/02/kafka-treasuredata-bridge-docker-compose/treasuredata.png)