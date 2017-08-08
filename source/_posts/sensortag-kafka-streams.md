title: "Kafka StreamsでSensorTagをウィンドウ集計をする"
date: 2017-08-08 10:44:19
categories:
 - IoT
tags:
 - Java
 - Kafka
 - KafkaStreams
description: PySpark StreamingでSensorTagのデータをJupyterを動作環境にしてウィンドウ集計を試しました。ストリーム処理のフレームワークは他にもいくつかありますが次はKafka Streamsを使ってみます。Sparkと違いこちらはクラスタではなくライブラリです。現在のところ開発言語は公式にはJavaのみサポートしています。
---

　PySpark StreamingでSensorTagのデータをJupyterを動作環境にして[ウィンドウ集計を試しました](https://masato.github.io/2017/08/02/sensortag-kafka-python-spark-streaming-6/)。ストリーム処理のフレームワークは他にもいくつかありますが次は[Kafka Streams](http://docs.confluent.io/current/streams/index.html)を使ってみます。Sparkと違いこちらはクラスタではなくライブラリです。現在のところ開発言語は公式にはJavaのみサポートしています。

<!-- more -->


## Java環境

　Ubuntu 16.04に構築した[Eclim](https://masato.github.io/2017/08/04/emacs-eclim-java/)をMavenでコードを書いていきます。


## プロジェクト

　プロジェクトのディレクトリに以下のファイルを作成します。完全なコードはこちらの[リポジトリ](https://github.com/masato/streams-kafka-examples)にあります。

```bash
$  tree
.
├── pom.xml
└── src
    └── main
        ├── java
        │   └── com
        │       └── github
        │           └── masato
        │               └── streams
        │                   └── kafka
        │                       ├── App.java
        │                       ├── SensorSumDeserializer.java
        │                       ├── SensorSum.java
        │                       └── SensorSumSerializer.java
        └── resources
            └── logback.xml

9 directories, 7 files
```


## App.java

　いくつかのパートにわけてコードの説明をします。

### 定数

　トピック名などはpom.xmlに定義した環境変数から取得します。`WINDOWS_MINUTES`はウィンドウ集計をする間隔です。`COMMIT_MINUTES`は後述するようにKafkaがキャッシュを自動的にコミットする間隔です。ここでは分で指定します。

```java
public class App {

    private static final Logger LOG = LoggerFactory.getLogger(App.class);

    private static final String SOURCE_TOPIC = System.getenv("SOURCE_TOPIC");
    private static final String SINK_TOPIC = System.getenv("SINK_TOPIC");
    private static final long WINDOWS_MINUTES = 2L;
    private static final long COMMIT_MINUTES = 3L;
```

### シリアライゼーション

　レコードのシリアライザとデシリアライザを作成します。Kafka Streamsアプリでは処理の中間結果をトピックに保存してフローを実装していきます。レコードをトピックから読むときのデシリアライザ、書くときのシリアライザの2つをまとめてSerDeを定義します。SerDeはトピックのキーと値の型ごとに必要です。

* jsonSerde
SensorTagのレコードはキーは文字列、値は[Jackson](http://wiki.fasterxml.com/JacksonHome)の[JsonNode](https://fasterxml.github.io/jackson-databind/javadoc/2.8/com/fasterxml/jackson/databind/JsonNode.html)オブジェクトです。

* sensorSumSerde
`SenroSum`はカスタムで作成した周囲温度 (ambient)とウィンドウ集計の状態を保持するクラスです。

* stringSerde
デフォルトのString用のSerDeです。今回メッセージのキーはすべて`String`です。

* doubleSerde
デフォルトのdouble用のSerDeです。SensorTagの周囲温度 (ambient)は`double`でウィンドウ集計します。

```java
    public static void main(String[] args) throws Exception {

        final Serializer<JsonNode> jsonSerializer = new JsonSerializer();
        final Deserializer<JsonNode> jsonDeserializer = new JsonDeserializer();
        final Serde<JsonNode> jsonSerde =
            Serdes.serdeFrom(jsonSerializer, jsonDeserializer);

        final Serializer<SensorSum> sensorSumSerializer =
            new SensorSumSerializer();
        final Deserializer<SensorSum> sensorSumDeserializer =
            new SensorSumDeserializer();
        final Serde<SensorSum> sensorSumSerde =
            Serdes.serdeFrom(sensorSumSerializer,
                             sensorSumDeserializer);

        final Serde<String> stringSerde = Serdes.String();
        final Serde<Double> doubleSerde = Serdes.Double();
```

### KStreamの作成

　最初に[KStreamBuilder](https://kafka.apache.org/0102/javadoc/org/apache/kafka/streams/kstream/KStreamBuilder.html)の`stream()`を呼び[KStream](https://kafka.apache.org/0102/javadoc/org/apache/kafka/streams/kstream/KStream.html)を作成します。トピックのキーは文字列、値はJsonNodeのSerDeを指定します。

```java
        final KStreamBuilder builder = new KStreamBuilder();

        LOG.info("Starting Sorting Job");

        final KStream<String, JsonNode> source =
            builder.stream(stringSerde, jsonSerde, SOURCE_TOPIC);
```

### KGroupedStreamを作成

　SensorTagのメッセージはRaspberry Pi 3からJSON文字列でKafkaのトピックに届きます。

```json
{'bid': 'B0:B4:48:BE:5E:00', 'time': 1502152524, 'humidity': 27.26287841796875, 'objecttemp': 21.1875, 'ambient': 27.03125, 'rh': 75.311279296875}
```

　KStreamのレコードはキーと値を持つ[KeyValue](https://kafka.apache.org/0102/javadoc/org/apache/kafka/streams/KeyValue.html)オブジェクトです。例では周囲温度 (ambient)の平均値だけウィンドウ集計するため`map()`を呼びキーと周囲温度のペアだけ持つ新しいKStreamを作成します。

　次に`groupByKey()`を呼びキーでグループ化して[KGroupedStream](https://kafka.apache.org/0102/javadoc/org/apache/kafka/streams/kstream/KGroupedStream.html)を作成します。レコードはキーは文字列、値は周囲温度の`double`になっているのでそれぞれのSerDeを指定します。

```java
        final KGroupedStream<String, Double> sensors =
            source
            .map((k, v) -> {
                    double ambient = v.get("ambient").asDouble();
                    return KeyValue.pair(k, ambient);
                })
            .groupByKey(stringSerde, doubleSerde);
```

### KStramからKTableを作成

　KGroupedStreamの`aggregate()`を呼び[KTable](https://kafka.apache.org/0102/javadoc/org/apache/kafka/streams/kstream/KTable.html)を作成します。KTableはキーごとに指定されたウィンドウ間隔でレコードの合計値とレコード数の状態を保持します。

　`aggregate()`の第1引数の[Initializer](https://kafka.apache.org/0102/javadoc/org/apache/kafka/streams/kstream/Initializer.html)ではストリームの集約で使うアグリゲータの初期化を行います。ここでウィンドウ集計の状態を保持する`SensorSum`の初期化を行います。第2引数でアグリゲータを実装します。現在のレコードのキーと値、1つ前のレコード処理で作成した`SensorSum`が渡されます。データの到着ごとに合計値とレコード数を加算して新しい`SensorSum`を返します。第３引数は2分ウィンドウの[TimeWindows](http://apache.mesi.com.ar/kafka/0.10.2.0/javadoc/org/apache/kafka/streams/kstream/TimeWindows.html)を定義します。第4引数は`SensorSum`のSerDe、第5引数は状態を保持するトピック名を渡します。

```java
        final KTable<Windowed<String>, SensorSum> sensorAgg =
            sensors
            .aggregate(() -> new SensorSum(0D, 0)
                       , (aggKey, value, agg) -> new SensorSum(agg.sum + value, agg.count + 1)
                       , TimeWindows.of(TimeUnit.MINUTES.toMillis(WINDOWS_MINUTES))
                       , sensorSumSerde,
                       "sensorSum");
```

### KTableからKStramを作成

　KTableの`mapValues()`で平均値を計算します。合計値をレコード数で除算した平均値は`Double`レコードの新しいKTableです。ここから`toStream()`を呼びKStreamを作成します。レコードはタイムスタンプ、キー、平均値のJSON文字列にフォーマットしてストリームに出力します。タイムスタンプは異なるシステム間でデータ交換がしやすいようにISO 8601にしています。最後に指定したトピックへレコードを保存して終了です。

```java
        final DateTimeFormatter fmt =
            DateTimeFormatter.ISO_OFFSET_DATE_TIME;

        sensorAgg
            .<Double>mapValues((v) -> ((double) v.sum / v.count))
            .toStream()
            .map((key, avg) -> {
                    long end = key.window().end();
                    ZonedDateTime zdt =
                        new Date(end).toInstant()
                        .atZone(ZoneId.systemDefault());
                    String time = fmt.format(zdt);
                    String bid = key.key();
                    String retval =
                        String.format("{\"time\": \"%s\", \"bid\": \"%s\", \"ambient\": %f}",
                                      time, bid, avg);
                    LOG.info(retval);
                    return new KeyValue<String,String>(bid, retval);
             })
            .to(SINK_TOPIC);
```

### Kafka Streamsの開始

　[KafkaStreams](https://kafka.apache.org/0102/javadoc/org/apache/kafka/streams/KafkaStreams.html)を設定オブジェクトとビルダーから作成してKafka Streamsアプリを開始します。また`SIGTERM`でKafka Streamを停止するようにシャットダウンフックに登録しておきます。

```java
        final StreamsConfig config = new StreamsConfig(getProperties());
        final KafkaStreams streams = new KafkaStreams(builder, config);
        streams.start();
        
        Runtime.getRuntime().addShutdownHook(new Thread(streams::close));
```

### Kafka Streamsの設定とタイムアウトについて

　環境変数などからKafka Streamsの設定で使う`Properties`を作成します。

```java
    private static Properties getProperties() {
        Properties props = new Properties();
        props.put(StreamsConfig.APPLICATION_ID_CONFIG,
                  System.getenv("APPLICATION_ID_CONFIG"));
        props.put(StreamsConfig.BOOTSTRAP_SERVERS_CONFIG,
                  System.getenv("BOOTSTRAP_SERVERS_CONFIG"));
        props.put(StreamsConfig.KEY_SERDE_CLASS_CONFIG,
                  Serdes.String().getClass().getName());
        props.put(StreamsConfig.VALUE_SERDE_CLASS_CONFIG,
                  Serdes.String().getClass().getName());
        props.put(StreamsConfig.TIMESTAMP_EXTRACTOR_CLASS_CONFIG,
                  WallclockTimestampExtractor.class.getName());
        props.put(StreamsConfig.COMMIT_INTERVAL_MS_CONFIG,
                  TimeUnit.MINUTES.toMillis(COMMIT_MINUTES));

        return props;
    }
```

### COMMIT_INTERVAL_MS_CONFIG

　最初は[StreamsConfig.COMMIT_INTERVAL_MS_CONFIG](https://kafka.apache.org/0102/javadoc/org/apache/kafka/streams/StreamsConfig.html#COMMIT_INTERVAL_MS_CONFIG)は変更していませんでした。レコードをトピック保存する前にKStreamのmap()でログを出力しています。2分ウィンドウ間隔の集計結果を最後に1回だけ出力をさせたかったのですが、4-5回不特定に重複する結果になりました。

```text
{"time": "2017-08-08T10:34:00+09:00", "bid": "B0:B4:48:BE:5E:00", "ambient": 27.343750}
{"time": "2017-08-08T10:34:00+09:00", "bid": "B0:B4:48:BE:5E:00", "ambient": 27.385417}
{"time": "2017-08-08T10:34:00+09:00", "bid": "B0:B4:48:BE:5E:00", "ambient": 27.410156}
{"time": "2017-08-08T10:34:00+09:00", "bid": "B0:B4:48:BE:5E:00", "ambient": 27.440341}
{"time": "2017-08-08T10:34:00+09:00", "bid": "B0:B4:48:BE:5E:00", "ambient": 27.450521}
{"time": "2017-08-08T10:36:00+09:00", "bid": "B0:B4:48:BE:5E:00", "ambient": 27.562500}
{"time": "2017-08-08T10:36:00+09:00", "bid": "B0:B4:48:BE:5E:00", "ambient": 27.562500}
```

　以下の記事を参考にすると、これはKTableの変更履歴 (changelog stream)という特徴から期待される動作のようです。KTableにウィンドウ集計の最終結果という状態はなく、キャッシュに更新された値は一定の間隔でコミットされます。KStreamへ`toStream()`したあとに`transform()`や`process()`を使いレコードの重複を除去するコードを自分で実装する必要があるようです。

　レコードの重複を完全に除去することはできませんが`StreamsConfig.COMMIT_INTERVAL_MS_CONFIG`の値を大きくすることでキャッシュがコミットされる回数を減らすことができます。[デフォルト値](http://docs.confluent.io/3.2.1/streams/developer-guide.html#optional-configuration-parameters)は30秒が指定されています。

* [How to send final kafka-streams aggregation result of a time windowed KTable?](https://stackoverflow.com/questions/38935904/how-to-send-final-kafka-streams-aggregation-result-of-a-time-windowed-ktable)
* [Immutable Record with Kafka Stream](http://users.kafka.apache.narkive.com/iFqyaD4p/immutable-record-with-kafka-stream)
* [Kafka KStreams - processing timeouts](https://stackoverflow.com/questions/39232395/kafka-kstreams-processing-timeouts)
* [Kafka Streams for Stream processing A few words about how Kafka works.](https://balamaci.ro/kafka-streams-for-stream-processing/)
* [Memory management](http://docs.confluent.io/3.1.2/streams/developer-guide.html#memory-management)

## その他のクラス

　モデル (SensorSum.java)、シリアライザ (SensorSumSerializer.java)、デシリアライザ (SensorSumDeserializer)のクラスを用意します。シリアライザは`serialize()`を実装して`SensorSum`のプロパティをバイト配列に変換します。byteバッファに周囲温度合計値の`Double`の8バイトと、レコード数の`Integer`の4バイト分を割り当て使います。

```java
    public byte[] serialize(String topic, SensorSum data) {
        ByteBuffer buffer = ByteBuffer.allocate(8 + 4);
        buffer.putDouble(data.sum);
        buffer.putInt(data.count);

        return buffer.array();
    }
```

## 実行

　[Exec Maven Plugin](http://www.mojohaus.org/exec-maven-plugin/)からKafka Streamsを実行します。

```bash
$ mvn clean install exec:exec@json
```

　ウィンドウ間隔が2分、キャッシュのコミット間隔を3分に指定してみました。やはり何回か重複した出力はありますが重複した出力を減らすことができました。

```text
{"time": "2017-08-08T11:32:00+09:00", "bid": "B0:B4:48:BE:5E:00", "ambient": 27.414773}
{"time": "2017-08-08T11:34:00+09:00", "bid": "B0:B4:48:BE:5E:00", "ambient": 27.414063}
{"time": "2017-08-08T11:36:00+09:00", "bid": "B0:B4:48:BE:5E:00", "ambient": 27.453125}
{"time": "2017-08-08T11:36:00+09:00", "bid": "B0:B4:48:BE:5E:00", "ambient": 27.476563}
{"time": "2017-08-08T11:38:00+09:00", "bid": "B0:B4:48:BE:5E:00", "ambient": 27.546875}
```
