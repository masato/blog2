title: "PythonとSensorTag, Kafka, Spark Streamingのストリーム処理 - Part 3: Apache AvroとSchema Registry"
date: 2017-07-30 12:46:11
categories:
 - IoT
tags:
 - Kafka
 - Landoop
 - SensorTag
 - Python
description: 　Landoopが提供するfast-data-devのDockerイメージにはSchema Registryも含まれています。前回はSensorTagのデータはJSONフォーマットで送信しましたがApache Avroフォーマットも試してみます。
---


<!-- more -->

　[Landoop](http://www.landoop.com)が提供する[fast-data-dev](https://hub.docker.com/r/landoop/fast-data-dev/)のDockerイメージには[Confluent Open Source](https://www.confluent.io/product/confluent-open-source/)の[Schema Registry](http://docs.confluent.io/current/schema-registry/docs/index.html)とWebツールの[Schema Registry UI](https://github.com/Landoop/schema-registry-ui)が含まれています。[前回](https://masato.github.io/2017/07/28/sensortag-kafka-python-spark-streaming-2/)SensorTagのデータはJSONフォーマットでKafkaへ送信していましたが[Apache Avro](https://avro.apache.org/)フォーマットも試してみます。Apache Avroはデータのシリアル化と言語に依存しないスキーマによるデータ交換の仕組みを提供します。Schema RegistryはREST APIから操作できるAvroスキーマを一元管理するためのストレージです。


## Schema Registry

　ローカルにあるAvroスキーマファイルを利用してデータをシリアライズすることもできますが、Schema Registryで一元管理することでAvroメッセージをデシリアライズする側も共通のデータフォーマットを参照することができます。


### Schema Registry UI

　fast-data-devのトップページからSCHEMASをクリックするとSchema Registry UIのページが開きます。左上にあるNEWボタンをクリックするとAvroスキーマを記述するエディタが起動します。


![schema-registry.png](/2017/07/30/sensortag-kafka-python-spark-streaming-3/schema-registry.png)


　[Schema Registry UI](https://github.com/Landoop/schema-registry-ui)のエディタでAvroスキーマを記述します。保存する前にバリデーションを実行するため記述したJSONが正しいフォーマットか確認できます。

　フォームの`Subject Name`はvalueスキーマの場合`topic名-value`と書くようです。SensorTagからAvroフォーマットで送信するtopic名は`sensortag-avro`なので、この場合は`sensortag-avro-value`になります。`Schema`のフィールドにSensorTag用のAvroスキーマをJSONフォーマットで記述します。


``` json
{
  "type": "record",
  "name": "SensorAvroValue",
  "fields": [
    {
      "name": "bid",
      "type": "string"
    },
    {
      "name": "time",
      "type": "long"
    },
    {
      "name": "ambient",
      "type": "double"
    },
    {
      "name": "objecttemp",
      "type": "double"
    },
    {
      "name": "humidity",
      "type": "double"
    },
    {
      "name": "rh",
      "type": "double"
    }
  ]
}
```


## Raspberry Pi 3

　[前回](https://masato.github.io/2017/07/28/sensortag-kafka-python-spark-streaming-2/)Raspberry Pi 3ではKafkaのPythonクライアントとして[kafka-python](http://kafka-python.readthedocs.io/en/master/)を利用しました。今回はAvroフォーマットに対応している[confluent-kafka-python](https://github.com/confluentinc/confluent-kafka-python)を使います。


### librdkafkaのインストール

　confluent-kafka-pythonのインストールには[librdkafka](https://github.com/edenhill/librdkafka)のヘッダが必要です。先にlibrdkafkaをビルドして共有ライブラリ情報を更新します。

```
$ sudo apt-get update && sudo apt-get install git build-essential -y
$ git clone https://github.com/edenhill/librdkafka.git
$ cd librdkafka
$ ./configure 
$ make && sudo make install
$ sudo ldconfig
```

### confluent-kafkaのインストール

　Pythonのヘッダファイルも必要です。Avroフォーマットを利用する場合pipパッケージ名は`confluent-kafka[avro]`になります。Avroが不要な場合は`confluent-kafka`です。

```
$ sudo apt-get update && sudo apt-get install python-dev -y
$ sudo pip install confluent-kafka[avro]
```

### Avro Producer

　オフィシャルの[confluent-kafka-python](https://github.com/confluentinc/confluent-kafka-python)のページにあるコードを参考にAvro Producerを書きます。公式サンプルではローカルにあるスキーマファイルを利用しています。スキーマをSchema Registryから取得する機能は実装されていないようなので、ちょっと手間ですがSchema Registryから直接REST APIでスキーマを文字列として取得します。


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

　SensorTagのBDアドレスをhcitoolを使い確認します。

```
$ sudo hcitool lescan
LE Scan ...
...
B0:B4:48:BE:5E:00 CC2650 SensorTag
...
```

　BDアドレスを引数にして作成したPythonスクリプトを実行します。

```
$ python avro_producer_sensortag.py <SensorTagのBDアドレス>
```

　以下のようなログを出力してKafkaブローカーへメッセージ送信を開始します。

```
{'bid': 'B0:B4:48:BE:5E:00', 'time': 1501495463, 'humidity': 27.04132080078125, 'objecttemp': 22.5, 'ambient': 26.84375, 'rh': 69.05517578125}
{'bid': 'B0:B4:48:BE:5E:00', 'time': 1501495475, 'humidity': 27.02117919921875, 'objecttemp': 22.75, 'ambient': 26.84375, 'rh': 69.05517578125}
{'bid': 'B0:B4:48:BE:5E:00', 'time': 1501495486, 'humidity': 27.04132080078125, 'objecttemp': 22.96875, 'ambient': 26.84375, 'rh': 69.05517578125}
```
　

### Avro Consumer

　Avro Consumerのコードは[confluent-kafka-python](https://github.com/confluentinc/confluent-kafka-python)にあるサンプルをそのまま使います。

``` python avro_consumer_sensortag.py
import requests
from confluent_kafka import KafkaError
from confluent_kafka.avro import AvroConsumer
from confluent_kafka.avro.serializer import SerializerError

c = AvroConsumer({
    'api.version.request':True,
    'bootstrap.servers': '<fast-data-devのIPアドレス>:9092',
    'group.id': 'raspiavro',
    'schema.registry.url': 'http://<fast-data-devのIPアドレス>:8081'})
c.subscribe(['sensortag-avro'])

running = True
while running:
    try:
        msg = c.poll(10)
        print(msg)
        if msg:
            if not msg.error():
                print(msg.value())
            elif msg.error().code() != KafkaError._PARTITION_EOF:
                print(msg.error())
                running = False
    except SerializerError as e:
        print("Message deserialization failed for %s: %s" % (msg, e))
        running = False

c.close()
```


　作成したPythoのスクリプトを実行します。

```
$ python avro_consumer_sensortag.py
```

　サンプルでは10秒間隔でpollingしています。タイミングがあわないとデータを取得できないためNoneが返ります。

```
<cimpl.Message object at 0x7655de88>
<cimpl.Message object at 0x764ee6f0>
{u'bid': u'B0:B4:48:BE:5E:00', u'time': 1501495204L, u'humidity': 27.27294921875, u'objecttemp': 22.78125, u'ambient': 27.09375, u'rh': 69.671630859375}
<cimpl.Message object at 0x7655de88>
None
<cimpl.Message object at 0x7655de88>
{u'bid': u'B0:B4:48:BE:5E:00', u'time': 1501495215L, u'humidity': 27.26287841796875, u'objecttemp': 22.9375, u'ambient': 27.09375, u'rh': 69.671630859375}
<cimpl.Message object at 0x747caa98>
```

### kafka-avro-console-consumer

　最後にサーバー側でも`kafka-avro-console-consumer`コマンドからメッセージを取得してみます。


```
$ docker-compose exec kafka-stack \
  kafka-avro-console-consumer \
  --bootstrap-server localhost:9092 \
  --topic sensortag-avro
```

　こちらも同様にSensorTagのデータを取得することができます。

```
{"bid":"B0:B4:48:BE:5E:00","time":1501495384,"ambient":26.9375,"objecttemp":22.96875,"humidity":27.11181640625,"rh":69.05517578125}
{"bid":"B0:B4:48:BE:5E:00","time":1501495396,"ambient":26.90625,"objecttemp":22.6875,"humidity":27.0916748046875,"rh":69.05517578125}
```
