title: "PythonとSensorTag, Kafka, Spark Streamingのストリーム処理 - Part 4: Kafka ConnectでMongoDBに出力する"
date: 2017-07-31 06:02:15
categories:
 - IoT
tags:
 - Kafka
 - Landoop
 - SensorTag
 - Python
description: Kafka ConnectはデータベースやKVSなど外部システムをKafkaに接続して連携させる仕組みです。スケールするストリーム処理のためのDataPipelineツールです。
---

　Kafka ConnectはデータベースやKVSなど外部システムをKafkaに接続して連携させる仕組みです。スケールするストリーム処理のためのDataPipelineツールです。ちょうどSensorTagのデータフォーマットを[Apache Avroに変更](https://masato.github.io/2017/07/30/sensortag-kafka-python-spark-streaming-3/)しました。[Kafka Connect UI](https://github.com/Landoop/kafka-connect-ui)ではデフォルトでAvroフォーマットを利用します。SensorTagのデータをKafkaのトピックを経由してMongoDBにSink(出力)してみます。


<!-- more -->


## Kafka Connect UI

　通常Kafka ConnectはCLIやREST APIを使い[Connector](http://docs.confluent.io/current/connect/managing.html)の設定を行います。[Kafka Connect UI](https://github.com/Landoop/kafka-connect-ui)の場合はエディタでConnectorの設定と[KCQL](https://github.com/datamountaineer/kafka-connect-query-language)を記述しConnectorを実行することができます。

### Connectors

　データベースなど外部システムをKafkaと接続するために、Source (入力)とSink (出力)の２種類のConnectorがあります。[デフォルト](http://docs.confluent.io/current/connect/connectors.html)でいくつかのConnectorは用意されています。Kafka Connect UIでは[demoページ](https://fast-data-dev.demo.landoop.com/kafka-connect-ui/#/cluster/fast-data-dev/select-connector)にあるように[Data Mountaineer](https://datamountaineer.com/)が開発しているConnectorが追加されています。
　
![kafka-connect-1.png](/2017/07/31/sensortag-kafka-python-spark-streaming-4/kafka-connect-1.png)
　

### KCQL

　[KCQL(Kafka Connect Query Language)](https://github.com/datamountaineer/kafka-connect-query-language)はSQL風にKafka ConnectのSourceとSinkの設定を記述することができます。例えばKafkaのtopicをMongoDBにSinkする場合[ドキュメント](http://docs.datamountaineer.com/en/latest/mongo-sink.html#starting-the-connector)にあるサンプルでは以下のように記述します。

```
name=mongo-sink-orders
connector.class=com.datamountaineer.streamreactor.connect.mongodb.sink.MongoSinkConnector
tasks.max=1
topics=orders-topic
connect.mongo.sink.kcql=INSERT INTO orders SELECT * FROM orders-topic
connect.mongo.database=connect
connect.mongo.connection=mongodb://localhost:27017
connect.mongo.sink.batch.size=10
```

## 使い方

　[fast-data-dev](https://github.com/Landoop/fast-data-dev)のdocker-compose.ymlを利用して構築したKafkaクラスタにMongoDBを追加してデータ連携のテストを行います。

* SensorTag -> Kafka -> Kafka Connect -> MongoDB

### MongoDB

　[fast-data-dev](https://github.com/Landoop/fast-data-dev)で構築したKafkaクラスタを起動しているdocker-compose.ymlにMongoDBサービスを追加します。

``` yaml docker-compose.yml
version: '2'
services:
  kafka-stack:
    image: landoop/fast-data-dev:latest
    environment:
      - FORWARDLOGS=0
      - RUNTESTS=0
      - ADV_HOST=<fast-data-devのIPアドレス>
    ports:
      - 3030:3030
      - 9092:9092
      - 2181:2181
      - 8081:8081
  connect-node-1:
    image: landoop/fast-data-dev-connect-cluster:latest
    depends_on:
      - kafka-stack
    environment:
      - ID=01
      - BS=kafka-stack:9092
      - ZK=kafka-stack:2181
      - SR=http://kafka-stack:8081
  connect-node-2:
    image: landoop/fast-data-dev-connect-cluster:latest
    depends_on:
      - kafka-stack
    environment:
      - ID=01
      - BS=kafka-stack:9092
      - ZK=kafka-stack:2181
      - SR=http://kafka-stack:8081
  connect-node-3:
    image: landoop/fast-data-dev-connect-cluster:latest
    depends_on:
      - kafka-stack
    environment:
      - ID=01
      - BS=kafka-stack:9092
      - ZK=kafka-stack:2181
      - SR=http://kafka-stack:8081
  connect-ui:
    image: landoop/kafka-connect-ui:latest
    depends_on:
      - connect-node-1
    environment:
      - CONNECT_URL=http://connect-node-1:8083
    ports:
      - 8000:8000
  mongo:
    image: mongo
    ports:
      - 27017:27017
    volumes:
      - mongo_data:/data/db
volumes:
  mongo_data:
    driver: local
```

　MongoDBサービスを起動します。

```
$ docker-compose up -d mongo
```


### Kafka Connect UI

　Kafka Connect UIのページを開きNEWボタン -> SinkからMongoDBを選択します。Create New Connector画面のエディタに以下のように記述してCreateボタンを押します。

```
name=MongoSinkConnector
connector.class=com.datamountaineer.streamreactor.connect.mongodb.sink.MongoSinkConnector
topics=sensortag-avro
tasks.max=1
connect.mongo.database=connect-db
connect.mongo.connection=mongodb://mongo:27017
connect.mongo.sink.kcql=INSERT INTO sensortag SELECT * FROM sensortag-avro
```

　サンプルから変更したところは以下です。

* connect.mongo.database

``` 
<MongoDBのデータベース名>
```

* connect.mongo.connection

```
mongodb://<MongoDBのサービス名>:27017
```

* connect.mongo.sink.kcql

```
INSERT INTO <MongoDBのコレクション名> SELECT * FROM <Kafkaのトピック名>
```

![kafka-connect-2.png](/2017/07/31/sensortag-kafka-python-spark-streaming-4/kafka-connect-2.png)


### Raspberry Pi 3

　Raspberry Pi 3にSSH接続します。[前回](https://masato.github.io/2017/07/30/sensortag-kafka-python-spark-streaming-3/)作成したAvroフォーマットでKafkaにデータ送信するPythonスクリプトを使います。
　

``` python avro_producer_sensortag.py
from bluepy.sensortag import SensorTag
import sys
import time
import calendar
import requests
from confluent_kafka import avro
from confluent_kafka.avro import AvroProducer

def main():
    argvs = sys.argv
    argc = len(argvs)
    if (argc != 2):
        print 'Usage: # python {0} bd_address'.format(argvs[0])
        quit()
    bid = argvs[1]
    print('Connecting to ' + bid)

    timeout = 10.0

    tag = SensorTag(bid)
    tag.IRtemperature.enable()
    tag.humidity.enable()

    time.sleep(1.0)

    get_schema_req_data = requests.get(
        "http://<fast-data-devのIPアドレス>:8081/subjects/sensortag-avro-value/versions/latest")
    get_schema_req_data.raise_for_status()

    schema_string = get_schema_req_data.json()['schema']
    value_schema = avro.loads(schema_string)

    avroProducer = AvroProducer({
        'api.version.request':True,
        'bootstrap.servers': '<fast-data-devのIPアドレス>:9092',
        'schema.registry.url': '<fast-data-devのIPアドレス>:8081'
    }, default_value_schema=value_schema)

    while True:
        tAmb, tObj = tag.IRtemperature.read()
        humidity, rh = tag.humidity.read()

        value = {
            "bid" : bid,
            "time" : calendar.timegm(time.gmtime()),
            "ambient": tAmb,
            "objecttemp": tObj,
            "humidity": humidity,
            "rh": rh
        }

        avroProducer.produce(topic='sensortag-avro', value=value)
        avroProducer.flush()
        print(value)
        time.sleep(timeout)

    tag.disconnect()
    del tag

if __name__ == '__main__':
    main()
```
　
　SensorTagのBDアドレスPythonスクリプトに渡して実行します。10秒間隔でKafkaの`sensortag-avro`トピックにAvroフォーマットのデータを送信します。

```
$ python avro_producer_sensortag.py <SensorTagのBDアドレス>
Connecting to B0:B4:48:BE:5E:00
{'bid': 'B0:B4:48:BE:5E:00', 'time': 1501541405, 'humidity': 26.9708251953125, 'objecttemp': 21.8125, 'ambient': 26.78125, 'rh': 73.62060546875}
{'bid': 'B0:B4:48:BE:5E:00', 'time': 1501541416, 'humidity': 26.990966796875, 'objecttemp': 22.625, 'ambient': 26.8125, 'rh': 73.52294921875}
```

### MongoDB

　MongoDBのコンテナに入りKafka Connect UIで設定したデータベースに接続します。KCQLの`INSERT INTO`に指定したコレクション(sensortag)にKafkaを経由してSensorTagのデータが出力されました。

```
$ docker-compose exec mongo mongo connect-db
> show collections;
sensortag
> db.sensortag.find()
{ "_id" : ObjectId("597fb4f4a7b11b00636cfc13"), "bid" : "B0:B4:48:BE:5E:00", "time" : NumberLong(1501541619), "ambient" : 26.96875, "objecttemp" : 22.9375, "humidity" : 27.152099609375, "rh" : 73.52294921875 }
{ "_id" : ObjectId("597fb4ffa7b11b00636cfc14"), "bid" : "B0:B4:48:BE:5E:00", "time" : NumberLong(1501541630), "ambient" : 26.96875, "objecttemp" : 22.9375, "humidity" : 27.1722412109375, "rh" : 73.431396484375 }
{ "_id" : ObjectId("597fb50ba7b11b00636cfc15"), "bid" : "B0:B4:48:BE:5E:00", "time" : NumberLong(1501541642), "ambient" : 27, "objecttemp" : 23.15625, "humidity" : 27.18231201171875, "rh" : 73.431396484375 }
```