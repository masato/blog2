title: "PythonとSensorTag, Kafka, Spark Streamingのストリーム処理 - Part 7: KafkaからTreasure Dataへ出力する"
date: 2017-08-02 10:24:03
categories:
 - IoT
tags:
 - Java
 - TreasureData
 - Kafka
description: 
---


　前回はJupyterからPySpark Streamingでウィンドウ集計した結果をKafkaのトピックに出力しました。このストリーム処理は分析ライフサイクルの前処理やエンリッチメントに相当します。パイプラインでは次にビッグデータのバッチ処理を仮定して[Treasure Data](https://www.treasuredata.co.jp/)に保存します。
　
<!-- more -->


## Docker Compose

```docker-compose.yml
services:
  td-agent:
    restart: always
    build: ./td-agent
    env_file:
      - ./.env
    ports:
      - 24224:24224
  kafka-bridge:
    restart: always
    build: ./kafka-bridge
    depends_on:
      - td-agent
```

## td-agent

### .env

```.env
TD_API_KEY=
TD_ENDPOINT=
```

### Dockerfile

```Dockerfile
FROM debian:8

RUN apt-get update && apt-get install -y curl

RUN curl https://packages.treasuredata.com/GPG-KEY-td-agent | apt-key add - && \
    echo "deb http://packages.treasuredata.com/2/debian/jessie/ jessie contrib" > /etc/apt/sources.list.d/treasure-data.list && \
    apt-get update && apt-get install -y td-agent && \
    apt-get clean && rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*

ADD td-agent.conf /etc/td-agent/
EXPOSE 24224
CMD ["/usr/sbin/td-agent"]
```

### td-agent.conf

```td-agent.conf
<match td.*.*>
  @type tdlog
  apikey "#{ENV['TD_API_KEY']}"
  endpoint "#{ENV['TD_ENDPOINT']}"
  auto_create_table
  buffer_type file
  buffer_path /var/log/td-agent/buffer/td
  <secondary>
    @type file
    path /var/log/td-agent/failed_records
  </secondary>
</match>

<source>
  @type forward
</source>
```

## kafka-fluentd-consumer


### Dockerfile

```Dockerfile
FROM java:8-jre
ARG KAFKA_FLUENTD_CONSUMER_VERSION=0.3.1

WORKDIR /app

RUN wget -q -O kafka-fluentd-consumer-all.jar https://github.com/treasure-data/kafka-fluentd-consumer/releases/download/v$KAFKA_FLUENTD_CONSUMER_VERSION/kafka-fluentd-consumer-$KAFKA_FLUENTD_CONSUMER_VERSION-all.jar

ADD log4j.properties .
ADD fluentd-consumer.properties .

CMD ["java", "-Dlog4j.configuration=file:///app/log4j.properties", "-jar", "kafka-fluentd-consumer-all.jar", "fluentd-consumer.properties"]
```

### fluentd-consumer.properties

```fluentd-consumer.properties
# Fluentd instance destinations.
fluentd.connect=td-agent:24224

# Dynamic event tag with topic name. 
fluentd.tag.prefix=td.gochikuru_dev.

# Consumed topics. 
fluentd.consumer.topics=gps

# The number of threads per consumer streams
fluentd.consumer.threads=1

# The path for backup un-flushed events during shutdown.
fluentd.consumer.backup.dir=/tmp/fluentd-consumer-backup/

# Kafka Consumer related parameters
zookeeper.connect=confluent:2181
group.id=test-consumer-group
zookeeper.session.timeout.ms=400
zookeeper.sync.time.ms=200
auto.commit.interval.ms=1000
```

### log4j.properties

```log4j.properties
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