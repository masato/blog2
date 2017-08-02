title: "PythonとSensorTag, Kafka, Spark Streamingのストリーム処理 - Part 8: KafkaからfreeboardにWebSocketでリアルタイムに出力する"
date: 2017-08-03 08:05:11
categories:
 - IoT
tags:
 - Kafka
 - freeboard
 - WebSocket
description: 
---

　PySpark Streamingでウィンドウ集計した結果は後続のバッチ処理のためTreasure Dataに保存しました。データ分析ライフサイクルの途中のデータもリアルタイムに確認できると、足りないデータや間違った結果のフィードバックをステークホルダーから早い段階で得られます。結果としてアジャイルなデータ分析アプリの開発につながると思います。

<!-- more -->

## freeboard

## kafka-proxy (Kafka -> WebSocket)

