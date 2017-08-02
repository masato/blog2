title: "PythonとSensorTag, Kafka, Spark Streamingのストリーム処理 - Part 6: JupyterからPySpark Streamingのウィンドウ集計をする"
date: 2017-08-02 08:01:51
categories:
 - IoT
tags:
 - Spark
 - Python
 - Kafka
 - Jupyter
description: ようやくタイトルのコードを実行する準備ができました。SensorTagのデータをKafkaに送信してPySpark Streamingのウィンドウ集計をします。JupyterをPythonのインタラクティブな実行環境に使います。
---

　ようやくタイトルのコードを実行する準備ができました。SensorTagのデータをKafkaに送信してPySpark Streamingのウィンドウ集計します。JupyterをPythonのインタラクティブな実行環境に使います。

## 準備

　これまでに用意したPythonスクリプトとKafka、Sparkのクラスタを使います。

* [Part 2](https://masato.github.io/2017/07/28/sensortag-kafka-python-spark-streaming-2/)で書いたRaspberry Pi 3からSensorTagのデータをJSONフォーマットでKafkaに送信するスクリプトと[Confluent Open Source](https://www.confluent.io/product/confluent-open-source/)クラスタ
* [Part 5](https://masato.github.io/2017/08/01/sensortag-kafka-python-spark-streaming-5/)で構築した[Spark Standalone Cluster](https://spark.apache.org/docs/2.1.1/spark-standalone.html)と[Jupyter](http://jupyter.org/)

## Notebook

　JupyterのNotebookにインタラクティブにコードを実行して確認していきます。以下のパラグラフはそれぞれセルに相当します。WebブラウザからJupyterを開き右上の`New`ボタンから`Python 3`を選択します。

* http://<仮想マシンのパブリックIPアドレス>:8888


### PYSPARK_SUBMIT_ARGS

　ScalaのJarファイルはバージョンの指定がかなり厳密です。[Spark Streaming + Kafka Integration Guide
](https://spark.apache.org/docs/2.1.1/streaming-kafka-integration.html)によるとSpark StreamingからKafkaに接続するためのJarファイルは2つあります。Kafkaのバージョンが`0.8.2.1`以上の[spark-streaming-kafka-0-8](https://spark.apache.org/docs/2.1.1/streaming-kafka-0-8-integration.html)と、`0.10`以上の[spark-streaming-kafka-0-10](https://spark.apache.org/docs/2.1.1/streaming-kafka-0-10-integration.html)です。今回利用しているKafkaのバージョンは`0.10.2.1`ですがPythonをサポートしている`spark-streaming-kafka-0-8`を使います。

　パッケージ名はspark-streaming-kafka-`<Kafkaのバージョン>`_`<Scalaのバージョン>`:`<Sparkのバージョン>`のようにそれぞれのバージョンを表しています。

```python
import os
os.environ['PYSPARK_SUBMIT_ARGS'] = '--packages org.apache.spark:spark-streaming-kafka-0-8_2.11:2.1.1 pyspark-shell'
```

### import

　Spark StreamingとKafkaに必要なパッケージをimportします。

```python
import json
import pytz
from datetime import datetime
from pyspark import SparkContext
from pyspark.streaming import StreamingContext
from pyspark.streaming.kafka import KafkaUtils
from pyspark.sql import SparkSession, Row
from pyspark.sql.functions import *
from pyspark.sql.types import *

from kafka import KafkaProducer
```

### Sparkのコンテキスト

　Spark 2.xになってからSparkのコンテキストがわかりにくくなっていますが、SparkSession.builderをエントリポイントにします。StreamingContextは1分のバッチ間隔で作成します。

```python
spark = (
    SparkSession
        .builder
        .getOrCreate()
)

sc = spark.sparkContext
ssc = StreamingContext(sc, 60)
```

### Kafka Producer

 Kafkaブローカーのリスト、入力と出力のトピック名を定義します。ウインドウ集計した結果はKafkaのトピックに出力するためproducerも作成します。

```python
brokers = "<Kafka BrokerのIPアドレス>:9092"
sourceTopic = "sensortag"
sinkTopic = "sensortag-sink"

producer = KafkaProducer(bootstrap_servers=brokers,
                         value_serializer=lambda v: json.dumps(v).encode('utf-8'))
```

### ウィンドウ集計

　今回のスクリプトのメインの処理をする関数です。KafkaのJSONを変換したRDDにStructTypeのスキーマを適用してDataFrameを作成します。DataFrameのウインドウ集計関数を使い2分ウインドウで周囲温度(ambient)と湿度(rh)の平均値を計算します。

　またDataFrameは処理の途中でタイムゾーンを削除しているためウィンドウ集計の結果がわかりやすいように`Asia/Tokyo`タイムゾーンをつけています。

```python
def windowAverage(rdd):
    schema = StructType([
        StructField('ambient', DoubleType(), True),
        StructField('bid', StringType(), True),
        StructField('humidity', DoubleType(), True),
        StructField('objecttemp', DoubleType(), True),
        StructField('rh', DoubleType(), True),
        StructField('time', TimestampType(), True),
    ])
        
    streamingInputDF = spark.createDataFrame(
        rdd, schema=schema
    )

    print('1分バッチのDataFrame')
    streamingInputDF.show(truncate=False)

    averageDF = (
        streamingInputDF
            .groupBy(
                streamingInputDF.bid,
                window("time", "2 minute"))
            .avg("ambient","rh")   
    )
    
    sinkRDD = averageDF.rdd.map(lambda x: {'bid': x[0], 
                                            'time': pytz.utc.localize(x[1]['end']).astimezone(pytz.timezone('Asia/Tokyo')).isoformat(), 
                                            'ambient': x[2], 
                                            'rh': x[3]})
    if not sinkRDD.isEmpty():
        print('2分ウィンドウの平均値')
        sinkList = sinkRDD.collect()
        print(sinkList)

        for sink in sinkList:
            producer.send(sinkTopic, sink)
```

### Kafkaのストリーム作成

　Kafka BrokerのIPアドレスとRaspberry Pi 3からSensorTagのJSON文字列を送信するトピックを指定します。JSON文字列を1行ずつデシリアライズして[pyspark.sql.Row](http://spark.apache.org/docs/2.1.1/api/python/pyspark.sql.html#pyspark.sql.Row)を作成します。`time`フィールドはUNIXタイムスタンプからPythonのdatetimeに変換しタイムゾーンを削除します。

```python
kafkaStream = KafkaUtils.createDirectStream(
    ssc, [sourceTopic], {"metadata.broker.list":brokers})

rowStream = (
    kafkaStream
        .map(lambda line: json.loads(line[1]))
        .map(lambda x: Row(
            ambient=x['ambient'],
            bid=x['bid'],
            humidity=x['humidity'],
            objecttemp=x['objecttemp'],
            rh=x['rh'],
            time=datetime.fromtimestamp(x['time']).replace(tzinfo=None),
        )
    )
)

rowStream.foreachRDD(windowAverage)
```

### StreamingContextの開始

　最後にStreamingContextを開始してプログラムが停止するまで待機します。

```python
ssc.start()
ssc.awaitTermination()
```

## スクリプトの実行

 Raspberry Pi 3から[Part 2](https://masato.github.io/2017/07/28/sensortag-kafka-python-spark-streaming-2/)で書いたPythonスクリプトを実行します。


### 出力結果

　以下の様な出力が表示されます。DataFrameの出力ではタイムゾーンがないのに対し、ウィンドウ集計の結果にはタイムゾーンが付与されています。

```bash
1分バッチのDataFrame
+--------+-----------------+-----------------+----------+---------------+---------------------+
|ambient |bid              |humidity         |objecttemp|rh             |time                 |
+--------+-----------------+-----------------+----------+---------------+---------------------+
|28.78125|B0:B4:48:BD:DA:03|28.72314453125   |22.96875  |75.714111328125|2017-08-01 23:44:03.0|
|28.78125|B0:B4:48:BD:DA:03|28.72314453125   |22.90625  |75.714111328125|2017-08-01 23:44:13.0|
|28.75   |B0:B4:48:BD:DA:03|28.72314453125   |22.875    |75.616455078125|2017-08-01 23:44:23.0|
|28.75   |B0:B4:48:BD:DA:03|28.69293212890625|23.15625  |75.616455078125|2017-08-01 23:44:34.0|
|28.75   |B0:B4:48:BD:DA:03|28.7030029296875 |23.03125  |75.616455078125|2017-08-01 23:44:44.0|
|28.75   |B0:B4:48:BD:DA:03|28.69293212890625|23.125    |75.616455078125|2017-08-01 23:44:55.0|
+--------+-----------------+-----------------+----------+---------------+---------------------+

2分ウィンドウの平均値
[{'bid': 'B0:B4:48:BD:DA:03', 'time': '2017-08-02T08:46:00+09:00', 'ambient': 28.760416666666668, 'rh': 75.64900716145833}]
```