title: "PythonとSensorTag, Kafka, Spark Streamingのストリーム処理 - Part 2: KafkaとLandoop"
date: 2017-07-28 12:24:03
categories:
 - IoT
tags:
 - Kafka
 - Landoop
 - SensorTag
 - Python
description: Landoopを使いKafkaクラスタをDocker Composeで構築します。
---

　[前回](https://masato.github.io/2017/07/27/sensortag-kafka-python-spark-streaming-1/)はRaspberry Pi 3上でSensorTagから環境データを取得するPythonスクリプトを書きました。この環境データはKafkaを経由してストリーム処理する予定です。次にRaspberry Pi 3からメッセージを受け取るKafkaクラスタをクラウド上に構築していきます。

　Kafkaクラスタは[Landoop](http://www.landoop.com)が開発している[fast-data-dev](https://hub.docker.com/r/landoop/fast-data-dev/)のDockerイメージを使います。
　

<!-- more -->

## LandoopでKafkaクラスタを構築する


　[Landoop](http://www.landoop.com/)はKafkaのソリューションを提供する会社です。特に[Kafka Connect](http://docs.confluent.io/current/connect/index.html)を中心にしたストリームのデータパイプラインの開発に強い印象です。

### Landoop　
　
　Landoopでは[Kafka Topics UI](http://kafka-topics-ui.landoop.com/#/)や[Kafka Connect UI](http://kafka-connect-ui.landoop.com/#/)、[Kafka Topics UI](https://hub.docker.com/r/landoop/kafka-topics-ui/)などのWebツールを開発しています。[demo](https://fast-data-dev.demo.landoop.com/)サイトからどのようなツールなのかを確認できます。このdemoサイトと同じ環境は[fast-data-connect-cluster](https://github.com/Landoop/fast-data-connect-cluster)を使うと簡単に構築することができます。
　

![landoop-top.png](/2017/07/28/sensortag-kafka-python-spark-streaming-2/landoop-top.png)


### fast-data-connect-cluster

　fast-data-connect-clusterのリポジトリをcloneします。

```
$ git clone https://github.com/Landoop/fast-data-connect-cluster
```
　
　リポジトリに含まれる[docker-compose.yml](https://github.com/Landoop/fast-data-connect-cluster/blob/master/docker-compose.yml)を少し変更してクラウド上で利用します。Debianの仮想マシンを用意してDockerとDocker Composeをインストールしました。利用するバージョンは以下です。

``` bash
$ docker --version
Docker version 17.06.0-ce, build 02c1d87

$ docker-compose --version
docker-compose version 1.14.0, build c7bdf9e
```

　このdocker-compose.ymlにはLandoopのWebツールに加えて[Confluent Open Source](https://www.confluent.io/product/confluent-open-source/)に含まれる[Kafka](https://kafka.apache.org/)、[Schema Registry](http://docs.confluent.io/current/schema-registry/docs/intro.html)、[Kafka REST Proxy](http://docs.confluent.io/current/kafka-rest/docs/index.html)、[Kafka Connect](http://docs.confluent.io/current/connect/index.html)、[Apache ZooKeeper](https://zookeeper.apache.org/)が含まれます。一通りKafkaを使った開発に必要なコンテナが揃うのでとても便利です。


　現在Confluent Open SourceとKafkaのバージョンは以下になっています。
　
* Confluent Open Source: v3.2.2
* Kafka v0.10.2.1


　docker-compose.ymlの主な変更点です。

* ADV_HOST: Dockerが起動している仮想マシンのパブリックIPアドレスを指定します。
* ports: リモートから接続するためにZooKeeperやKafkaクラスタのポートを公開します。


``` yaml docker-compose.yml
version: '2'
services:
  kafka-stack:
    image: landoop/fast-data-dev
    environment:
      - FORWARDLOGS=0
      - RUNTESTS=0
      - ADV_HOST=210.xxx.xxx.xxx
    ports:
      - 3030:3030
      - 9092:9092
      - 2181:2181
      - 8081:8081
  connect-node-1:
    image: landoop/fast-data-dev-connect-cluster
    depends_on:
      - kafka-stack
    environment:
      - ID=01
      - BS=kafka-stack:9092
      - ZK=kafka-stack:2181
      - SR=http://kafka-stack:8081
  connect-node-2:
    image: landoop/fast-data-dev-connect-cluster
    depends_on:
      - kafka-stack
    environment:
      - ID=01
      - BS=kafka-stack:9092
      - ZK=kafka-stack:2181
      - SR=http://kafka-stack:8081
  connect-node-3:
    image: landoop/fast-data-dev-connect-cluster
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
```


　docker-compose.ymlのディレクトリに移動してコンテナを起動します。

```
$ cd fast-data-connect-cluster
$ docker-compose up -d
```


### 動作確認

　Kafkaクライアントから簡単に動作確認をします。kafka-topicsコマンドでトピックを作成します。

``` bash
$ docker-compose exec kafka-stack kafka-topics \
    --create --topic test \
    --zookeeper localhost:2181 \
    --partitions 1 --replication-factor 1
```


　kafka-console-consumerコマンドを実行します。メッセージがトピックに届くとこのシェルに表示されます。

``` bash
$ docker-compose exec kafka-stack \
  kafka-console-consumer \
  --bootstrap-server localhost:9092 \
  --from-beginning \
  --topic test
```

　別のシェルからkafka-console-producerコマンドを実行します。

``` bash
$ docker-compose exec kafka-stack \
  kafka-console-producer \
  --broker-list localhost:9092 \
  --topic test
```

　コマンドは待機状態になるので適当なメッセージをシェルに入力します。同じメッセージがkafka-console-consumerのシェルにも表示されます。

``` bash
hello world
```

　Kafka Topics UIのページではトピックの一覧とメッセージの中身を確認することができます。
　
![landoop-topic.png](/2017/07/28/sensortag-kafka-python-spark-streaming-2/landoop-topic.png)


## SensorTagの環境データをKafkaに送信する


### kafka-python

　PythonのKafkaクライアントには[kafka-python](http://kafka-python.readthedocs.io/en/master/)と[confluent-kafka-python](http://docs.confluent.io/current/clients/confluent-kafka-python/)があります。APIが微妙に違うので間違えないようにします。今回はkafka-pythonをインストールして使います。

``` bash
$ sudo pip install kafka-python
```

　[前回](https://masato.github.io/2017/07/27/sensortag-kafka-python-spark-streaming-1/)書いたSensorTagのデータを取得するPythonのコードにKafkaへメッセージを送信するproducerを追加します。

``` python json_producer_sensortag_kafka.py
from bluepy.sensortag import SensorTag
import sys
import time
import json
import calendar

from kafka import KafkaProducer

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

    producer = KafkaProducer(bootstrap_servers='210.xxx.xxx.xxx:9092')
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

        msg = json.dumps(value).encode("utf-8")
        producer.send('sensortag', msg)
        producer.flush()
        print(msg)

        time.sleep(timeout)

    tag.disconnect()
    del tag

if __name__ == '__main__':
    main()
```


　SensorTagのBDアドレスを引数にしてPythonスクリプトを実行します。


``` bash
$ python json_producer_sensortag_kafka.py B0:B4:48:BE:5E:00
```


　10秒間隔で取得した環境データをJSON文字列に整形してKafkaクラスタに送信します。


``` bash
Connecting to B0:B4:48:BE:5E:00
{"bid": "B0:B4:48:BE:5E:00", "time": 1501464133, "humidity": 26.8096923828125, "objecttemp": 22.0625, "ambient": 26.59375, "rh": 68.829345703125}
{"bid": "B0:B4:48:BE:5E:00", "time": 1501464143, "humidity": 26.86004638671875, "objecttemp": 22.40625, "ambient": 26.65625, "rh": 68.927001953125}
{"bid": "B0:B4:48:BE:5E:00", "time": 1501464153, "humidity": 26.92047119140625, "objecttemp": 22.71875, "ambient": 26.71875, "rh": 68.95751953125}
```


　Kafka Topics UIの画面にもSensorTagの環境データが表示されました。

![landoop-sensortag.png](/2017/07/28/sensortag-kafka-python-spark-streaming-2/landoop-sensortag.png)
