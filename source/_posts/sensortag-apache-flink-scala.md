title: "Apache FlinkとScalaでセンサーデータをウィンドウ集計をする"
date: 2017-08-10 14:38:25
categories:
 - IoT
tags:
 - SensorTag
 - Scala
 - ApacheFlink
description: Spark Streaming、Kafka Streamsに続いてストリーム処理フレームワークのApache Flinkを試します。Spark StreamingはPython、Kafka StreamsはJavaで書いたのでApache FlinkはScalaで書いてみようと思います。
---

　[Spark Streaming](https://masato.github.io/2017/08/02/sensortag-kafka-python-spark-streaming-6/)、[Kafka Streams](https://masato.github.io/2017/08/08/sensortag-kafka-streams/)に続いてストリーム処理フレームワークの[Apache Flink](https://flink.apache.org/)を試します。Spark StreamingはPython、Kafka StreamsはJavaで書いたのでApache FlinkはScalaで書いてみようと思います。

<!-- more -->

　Apache FlinkもKafkaと同様にScalaで書かれています。Scalaに特徴的な後方互換性を重視せずアグレッシブな開発をしています。そのためネットで検索できる情報もどんどん古くなりAPIもDeprecatedやPublicEvolvingになりがちで初学者には少し入りづらい状況です。なかなか学習用の良い記事が見つかりませんでしたが、センサーデータのウィンドウ集計の書き方は[THE RISE OF BIG DATA STREAMING](http://blog.scottlogic.com/2017/02/07/the-rise-of-big-data-streaming.html)を参考にさせていただきました。

## プロジェクトテンプレート

　Apache Flinkのプロジェクトは[A flink project template using Scala and SBT](https://github.com/tillrohrmann/flink-project)をテンプレートにすると便利です。最初にテンプレートのWordCountを例に使い方を確認します。テンプレートをcloneします。

```
$ cd ~/scala_apps
$ git clone https://github.com/tillrohrmann/flink-project.git
```

　いくつか例がありますがここではWordCount.scalaを使います。

```
$ tree flink-project
flink-project/
├── build.sbt
├── idea.sbt
├── project
│   ├── assembly.sbt
│   └── build.properties
├── README
└── src
    └── main
        ├── resources
        │   └── log4j.properties
        └── scala
            └── org
                └── example
                    ├── Job.scala
                    ├── SocketTextStreamWordCount.scala
                    └── WordCount.scala

```

　[ENSIME](http://ensime.org/)を使う場合は[こちら](https://masato.github.io/2017/08/09/emacs-ensime-sdkman/)を参考にしてプロジェクトに`.ensime`ファイルを作成してEmacsから`M-x ensime`してください。

```
$ cd ~/scala_apps/flink-project
$ sbt
> ensimeConfig
```

　WordCount.scalaのコードです。例のテキストに含まれる英単語をカウントします。

```scala WordCount.scala
package org.example
import org.apache.flink.api.scala._

object WordCount {
  def main(args: Array[String]) {

    val env = ExecutionEnvironment.getExecutionEnvironment

    val text = env.fromElements("To be, or not to be,--that is the question:--",
      "Whether 'tis nobler in the mind to suffer", "The slings and arrows of outrageous fortune",
      "Or to take arms against a sea of troubles,")

    val counts = text.flatMap { _.toLowerCase.split("\\W+") }
      .map { (_, 1) }
      .groupBy(0)
      .sum(1)

    counts.print()

  }
}
```

　プロジェクトのディレクトリからsbtのrunコマンドを実行します。mainメソッドを実装したクラスがいくつかあるのでWordCountの`3`を入力します。

```
$ cd ~/scala_apps/flink-project
$ sbt
> run
Multiple main classes detected, select one to run:

 [1] org.example.Job
 [2] org.example.SocketTextStreamWordCount
 [3] org.example.WordCount

Enter number:3
```

　実行すると以下のようにテキストに含まれる英単語を数えて出力します。

```
 (a,1)
 (fortune,1)
 (in,1)
 (mind,1)
 (or,2)
 (question,1)
 (slings,1)
 (suffer,1)
 (take,1)
 (that,1)
 (to,4)
```

　テキストデータは[ExecutionEnvironment](https://ci.apache.org/projects/flink/flink-docs-release-1.3/api/java/org/apache/flink/api/scala/ExecutionEnvironment.html)から[fromElements](https://ci.apache.org/projects/flink/flink-docs-release-1.3/api/java/org/apache/flink/api/scala/ExecutionEnvironment.html#fromElements-scala.collection.Seq-scala.reflect.ClassTag-org.apache.flink.api.common.typeinfo.TypeInformation-)して作成した[DataSource](https://ci.apache.org/projects/flink/flink-docs-release-1.3/api/java/org/apache/flink/api/java/operators/DataSource.html)です。

```scala
    val env = ExecutionEnvironment.getExecutionEnvironment

    val text = env.fromElements("To be, or not to be,--that is the question:--",
      "Whether 'tis nobler in the mind to suffer", "The slings and arrows of outrageous fortune",
      "Or to take arms against a sea of troubles,")
```

　Apach FlinkのScalaではコードを短く書ける反面でアンダースコアや`map`や`groupBy`に登場する1や0が何を指しているのかわかりにくいことがあります。Apache FlinkのTupleはfieldで指定する場合は`zero indexed`なので順番に`0, 1`となります。

```scala
    val counts = text.flatMap { _.toLowerCase.split("\\W+") }
      .map { (_, 1) }
      .groupBy(0)
      .sum(1)
```

　テキストデータを`flatMap`で正規表現を使い単語に区切り[DataSet](https://ci.apache.org/projects/flink/flink-docs-release-1.3/api/java/org/apache/flink/api/scala/DataSet.html)を作成します。[map()](https://ci.apache.org/projects/flink/flink-docs-release-1.3/api/java/org/apache/flink/api/scala/DataSet.html#map-scala.Function1-org.apache.flink.api.common.typeinfo.TypeInformation-scala.reflect.ClassTag-)で単語(`_`)と数(`1`)でTupleの[DataSet](https://ci.apache.org/projects/flink/flink-docs-release-1.3/api/java/org/apache/flink/api/scala/DataSet.html)を作成します。[groupBy()](https://ci.apache.org/projects/flink/flink-docs-release-1.3/api/java/org/apache/flink/api/scala/DataSet.html#groupBy-scala.collection.Seq-)はfieldの`0`を指定して単語でグループ化した[GroupedDataSet](https://ci.apache.org/projects/flink/flink-docs-release-1.3/api/java/org/apache/flink/api/scala/GroupedDataSet.html)を作成します。最後に[sum()](https://ci.apache.org/projects/flink/flink-docs-release-1.3/api/java/org/apache/flink/api/scala/GroupedDataSet.html#sum-int-)の引数にfieldの`1`を指定して単語と単語の合計数をTupleにした[AggregateDataSet](https://ci.apache.org/projects/flink/flink-docs-release-1.3/api/java/org/apache/flink/api/scala/AggregateDataSet.html)を作成します。


## ウィンドウ集計

　このテンプレートプロジェクトを使いセンサーデータを60秒のタンブリングウィンドウで集計して周囲温度(ambient)の平均値を計算するプログラムを書きます。KafkaをSourceにするので[こちら](https://masato.github.io/2017/07/28/sensortag-kafka-python-spark-streaming-2/)を参考にRaspberry Pi 3からSensorTagのデータをKafkaに送信します。

```
Raspberry Pi 3 -> Source (Kafka) -> ストリーム処理 -> Sink (Kafka)
```

　テンプレートプロジェクトをcloneした後に既存のファイルを削除します。

```
$ cd ~/scala_apps
$ git clone https://github.com/tillrohrmann/flink-project.git streams-flink-scala-examples
$ cd streams-flink-scala-examples
$ rm -fr src/main/scala/org/
```

　Scalaのパッケージディレクトリを作成します。

```
$ mkdir -p src/main/scala/com/github/masato/streams/flink
```

### build.sbt

　Kafkaは[こちら](https://masato.github.io/2017/08/02/kafka-treasuredata-bridge-docker-compose/)と同様に[landoop/fast-data-dev](https://hub.docker.com/r/landoop/fast-data-dev/)のDockerイメージを利用します。バージョンは`0.10.2.1`です。Kafka 0.10に対応したパッケージを追加します。


```scala build.sbt
val flinkVersion = "1.3.2"
 
val flinkDependencies = Seq(
  "org.apache.flink" %% "flink-scala" % flinkVersion % "provided",
  "org.apache.flink" %% "flink-streaming-scala" % flinkVersion % "provided",
  "org.apache.flink" %% "flink-connector-kafka-0.10" % flinkVersion)

val otherDependencies = Seq(
   "com.typesafe" % "config" % "1.3.1"
)

lazy val root = (project in file(".")).
  settings(
    libraryDependencies ++= flinkDependencies,
    libraryDependencies ++= otherDependencies
  )
   
mainClass in assembly := Some("com.github.masato.streams.flink.App")
```

### App.scala

　メインメソッドを実装したプログラムの全文です。Kafkaへの接続情報などは[config](https://typesafehub.github.io/config/)を使い設定ファイルに定義します。ソースコードは[リポジトリ](https://github.com/masato/streams-flink-scala-examples)にもあります。

```scala App.scala
package com.github.masato.streams.flink

import java.util.Properties
import java.time.ZoneId;
import java.time.format.DateTimeFormatter
import java.util.Date

import org.apache.flink.streaming.api.TimeCharacteristic
import org.apache.flink.streaming.api.scala._
import org.apache.flink.streaming.api.functions.IngestionTimeExtractor
import org.apache.flink.streaming.api.functions.timestamps.BoundedOutOfOrdernessTimestampExtractor
import org.apache.flink.streaming.api.windowing.time.Time
import org.apache.flink.streaming.api.windowing.windows.TimeWindow
import org.apache.flink.streaming.connectors.kafka.{FlinkKafkaConsumer010,FlinkKafkaProducer010}
import org.apache.flink.streaming.util.serialization.{JSONDeserializationSchema,SimpleStringSchema}
import org.apache.flink.api.common.functions.AggregateFunction

import org.apache.flink.util.Collector
import com.fasterxml.jackson.databind.node.ObjectNode
import scala.util.parsing.json.JSONObject
import com.typesafe.config.ConfigFactory

case class Accumulator(time: Long, bid: String, var sum: Double, var count: Int)

class Aggregate extends AggregateFunction[(String, Double), Accumulator,Accumulator] {

  override def createAccumulator(): Accumulator = {
    return Accumulator(0L, "", 0.0, 0)
  }

  override def merge(a: Accumulator, b: Accumulator): Accumulator = {
    a.sum += b.sum
    a.count += b.count
    return a
  }

  override def add(value: (String, Double), acc: Accumulator): Unit = {
    acc.sum += value._2
    acc.count += 1
  }

  override def getResult(acc: Accumulator): Accumulator = {
    return acc
  }
}

object App {
  val fmt = DateTimeFormatter.ISO_OFFSET_DATE_TIME

  val conf = ConfigFactory.load()
  val bootstrapServers = conf.getString("app.bootstrap-servers")
  val groupId = conf.getString("app.group-id")
  val sourceTopic = conf.getString("app.source-topic")
  val sinkTopic = conf.getString("app.sink-topic")

  def main(args: Array[String]): Unit = {
    val props = new Properties()
    props.setProperty("bootstrap.servers", bootstrapServers)
    props.setProperty("group.id", groupId)

    val env = StreamExecutionEnvironment.getExecutionEnvironment
    env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime)

    val source = new FlinkKafkaConsumer010[ObjectNode](
      sourceTopic, new JSONDeserializationSchema(), props)

    val events = env.addSource(source).name("events")

    val timestamped = events.assignTimestampsAndWatermarks(
      new BoundedOutOfOrdernessTimestampExtractor[ObjectNode](Time.seconds(10)) {
        override def extractTimestamp(element: ObjectNode): Long = element.get("time").asLong * 1000
      })

    timestamped
      .map { v =>
        val key =  v.get("bid").asText
        val ambient = v.get("ambient").asDouble
        (key, ambient)
      }
      .keyBy(v => v._1)
      .timeWindow(Time.seconds(60))
      .aggregate(new Aggregate(),
        ( key: String,
          window: TimeWindow,
          input: Iterable[Accumulator],
          out: Collector[Accumulator] ) => {
            var in = input.iterator.next()
            out.collect(Accumulator(window.getEnd, key, in.sum/in.count, in.count))
          }
      )
      .map { v =>
        val zdt = new Date(v.time).toInstant().atZone(ZoneId.systemDefault())
        val time = fmt.format(zdt)
        val json = Map("time" -> time, "bid" -> v.bid, "ambient" -> v.sum)
        val retval = JSONObject(json).toString()
        println(retval)
        retval
      }
      .addSink(new FlinkKafkaProducer010[String](
        bootstrapServers,
        sinkTopic,
        new SimpleStringSchema)
      ).name("kafka")
    env.execute()
  }
}
```

　main()の処理を順番にみていきます。最初にKafkaに接続する設定を行います。接続するKafkaが0.10のため[FlinkKafkaConsumer010](https://ci.apache.org/projects/flink/flink-docs-release-1.3/api/java/org/apache/flink/streaming/connectors/kafka/FlinkKafkaConsumer010.html)を使います。Raspberry Pi 3から届くセンサーデータは以下のようなJSONフォーマットです。

```
{'bid': 'B0:B4:48:BE:5E:00', 'time': 1503527847, 'humidity': 26.55792236328125, 'objecttemp': 22.3125, 'ambient': 26.375, 'rh': 76.983642578125}
```

　[JSONDeserializationSchema](https://ci.apache.org/projects/flink/flink-docs-release-1.3/api/java/org/apache/flink/streaming/util/serialization/JSONDeserializationSchema.html)でデシリアライズします。

```scala
    val props = new Properties()
    props.setProperty("bootstrap.servers", bootstrapServers)
    props.setProperty("group.id", groupId)

    val env = StreamExecutionEnvironment.getExecutionEnvironment
    env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime)

    val source = new FlinkKafkaConsumer010[ObjectNode](
      sourceTopic, new JSONDeserializationSchema(), props)
```

　Apache Flinkの時間モデルはイベント時間 (TimeCharacteristic.EventTime)に設定しています。センサーデータの`time`フィールドをタイムスタンプとウォーターマークに使います。

```scala
    val events = env.addSource(source).name("events")

    val timestamped = events.assignTimestampsAndWatermarks(
      new BoundedOutOfOrdernessTimestampExtractor[ObjectNode](Time.seconds(10)) {
        override def extractTimestamp(element: ObjectNode): Long = element.get("time").asLong * 1000
      })
```



　センサーからはいくつかのデータが取得できていますがここでは周囲温度(ambient)の値だけ利用します。[map()](https://ci.apache.org/projects/flink/flink-docs-release-1.3/api/java/org/apache/flink/api/scala/DataSet.html#map-scala.Function1-org.apache.flink.api.common.typeinfo.TypeInformation-scala.reflect.ClassTag-)でSensorTagのBDアドレスをキーにして新しいTupleを作成します。

```scala
    timestamped
      .map { v =>
        val key =  v.get("bid").asText
        val ambient = v.get("ambient").asDouble
        (key, ambient)
      }
```

　[DataStream](https://ci.apache.org/projects/flink/flink-docs-release-1.3/api/java/org/apache/flink/streaming/api/datastream/DataStream.html)の[keyBy()](https://ci.apache.org/projects/flink/flink-docs-release-1.3/api/java/org/apache/flink/streaming/api/scala/DataStream.html#keyBy-scala.collection.Seq-)にTupleのインデックス`1`を指定してBDアドレスをキーにした[KeyedStream](https://ci.apache.org/projects/flink/flink-docs-release-1.3/api/java/org/apache/flink/streaming/api/scala/KeyedStream.html)を作成します。

```scala
      .keyBy(v => v._1)
```

　[timeWindow()](https://ci.apache.org/projects/flink/flink-docs-release-1.3/api/java/org/apache/flink/streaming/api/datastream/KeyedStream.html#timeWindow-org.apache.flink.streaming.api.windowing.time.Time-)で60秒に設定したタンブリングウィンドウの[WindowedStream](https://ci.apache.org/projects/flink/flink-docs-release-1.3/api/java/org/apache/flink/streaming/api/datastream/WindowedStream.html)を作成します。

```scala
      .timeWindow(Time.seconds(60))
```

　バージョン1.3では[apply()](https://ci.apache.org/projects/flink/flink-docs-release-1.3/api/java/org/apache/flink/streaming/api/datastream/WindowedStream.html#apply-R-org.apache.flink.api.common.functions.FoldFunction-org.apache.flink.streaming.api.functions.windowing.WindowFunction-)はDeprecatedになっています。以前は以下のように書けました。

```scala
      .apply(
        (0L, "", 0.0, 0),
        (acc: (Long, String, Double, Int),
         v: (String, Double)) => { (0L, v._1, acc._3 + v._2, acc._4 + 1) },
        ( window: TimeWindow,
          counts: Iterable[(Long, String, Double, Int)],
          out: Collector[(Long, String, Double, Int)] ) => {
            var count = counts.iterator.next()
            out.collect((window.getEnd, count._2, count._3/count._4, count._4))
          }
      )
```

　さらに[fold()](https://ci.apache.org/projects/flink/flink-docs-release-1.3/api/java/org/apache/flink/streaming/api/datastream/WindowedStream.html#fold-ACC-org.apache.flink.api.common.functions.FoldFunction-org.apache.flink.streaming.api.functions.windowing.WindowFunction-)もDeprecatedなので推奨されている[aggregate()](https://ci.apache.org/projects/flink/flink-docs-release-1.3/api/java/org/apache/flink/streaming/api/datastream/WindowedStream.html#aggregate-org.apache.flink.api.common.functions.AggregateFunction-org.apache.flink.streaming.api.functions.windowing.WindowFunction-)を使ってみます。`Aggregate`は[AggregateFunction](https://ci.apache.org/projects/flink/flink-docs-release-1.3/api/java/org/apache/flink/api/common/functions/AggregateFunction.html)を実装しています。`apply()`の例のようにTupleを使っても良いですがcaseクラスにすると少し読みやすくなります。

```scala
      .aggregate(new Aggregate(),
        ( key: String,
          window: TimeWindow,
          input: Iterable[Accumulator],
          out: Collector[Accumulator] ) => {
            var in = input.iterator.next()
            out.collect(Accumulator(window.getEnd, key, in.sum/in.count, in.count))
          }
      )
```

　外部システムと連携しやすいようにデータストリームはUNIXタイムはタイムゾーンをつけたISO-8601にフォーマットしたJSON文字列に[map()](https://ci.apache.org/projects/flink/flink-docs-release-1.3/api/java/org/apache/flink/streaming/api/datastream/DataStream.html#map-org.apache.flink.api.common.functions.MapFunction-)します。ここではデバッグ用にJSON文字列は標準出力しています。

```scala
      .map { v =>
        val zdt = new Date(v.time).toInstant().atZone(ZoneId.systemDefault())
        val time = fmt.format(zdt)
        val json = Map("time" -> time, "bid" -> v.bid, "ambient" -> v.sum)
        val retval = JSONObject(json).toString()
        println(retval)
        retval
      }
```

　最後にデータストリームをKafkaにSinkします。`name("kafka")`のようにSinkに名前をつけると実行時のログに表示されます。

```scala
      .addSink(new FlinkKafkaProducer010[String](
        bootstrapServers,
        sinkTopic,
        new SimpleStringSchema)
      ).name("kafka")
    env.execute()
```

### sbtのrun

　プロジェクトに移動してsbtのrunコマンドを実行します。

```
$ cd ~/scala_apps/streams-flink-scala-examples
$ sbt
> run
```

　周囲温度(ambient)を60秒のタンブリングウィンドウで集計した平均値が標準出力されました。

```
{"time" : "2017-08-24T08:10:00+09:00", "bid" : "B0:B4:48:BE:5E:00", "ambient" : 26.203125}
{"time" : "2017-08-24T08:11:00+09:00", "bid" : "B0:B4:48:BE:5E:00", "ambient" : 26.234375}
{"time" : "2017-08-24T08:12:00+09:00", "bid" : "B0:B4:48:BE:5E:00", "ambient" : 26.26875}
```