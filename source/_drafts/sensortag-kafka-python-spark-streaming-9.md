title: "PythonとSensorTag, Kafka, Spark Streamingのストリーム処理 - Part 9: Raspberry Pi 3からKafka REST Proxyへデータ送信する"
date: 2017-08-04 10:39:08
categories:
 - IoT
tags:
 - Kafka
 - freeboard
 - WebSocket
description: 
---

 これまでSensorTagで取得したデータはRaspberry Pi 3からPythonのKafkaクライアントを使い送信していました。Linuxの場合はKafkaクライアントがあるので便利ですが、その他の限定されたデバイスでネイティブのKafkaのプロトコルを実装するのは大変です。Kafka REST ProxyはKafkaクラスタへのREST APIを提供します。HTTP/HTTPSが使える環境ならより簡単にKafkaとのメッセージの入出力を行えます。


<!-- more -->


## 

## freeboard

　前回作成したfreeboardとkafka-proxy (Kafka -> WebSocket)のコンテナ