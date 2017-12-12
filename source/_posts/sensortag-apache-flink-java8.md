title: "Apache FlinkとJava 8でセンサーデータをウィンドウ集計をする"
date: 2017-08-27 14:13:01
categories:
 - IoT
tags:
 - SensorTag
 - Scala
 - ApacheFlink
description: SensorTagのセンサーデータをApache FlinkとScala APIを使いウィンドウ集計を試しました。Scala APIとなるべく同じようにJava 8 APIで書き直します。
---

　SensorTagのセンサーデータをApache FlinkとScala APIを使いウィンドウ集計を[試しました](https://masato.github.io/2017/08/10/sensortag-apache-flink-scala/)。[Scala API](https://ci.apache.org/projects/flink/flink-docs-release-1.3/dev/scala_api_extensions.html)となるべく同じように[Java 8 API](https://ci.apache.org/projects/flink/flink-docs-release-1.3/dev/java8.html)で書き直します。


<!-- more -->

## Mavenアーキタイプ

　[Sample Project using the Java API](https://ci.apache.org/projects/flink/flink-docs-release-1.3/quickstart/java_api_quickstart.html)にある[flink-quickstart-java](https://mvnrepository.com/artifact/org.apache.flink/flink-quickstart-java)を使いMavenプロジェクトを作成します。Apache FlinkのバージョンはScalaの時と同じ`1.3.2`です。`groupId`や`package`は環境にあわせて変更してください。

```bash
$ mkdir -p ~/java_apps && cd ~/java_apps
$ mvn archetype:generate \
    -DarchetypeGroupId=org.apache.flink \
    -DarchetypeArtifactId=flink-quickstart-java \
    -DarchetypeVersion=1.3.2 \
    -DgroupId=streams-flink-java-examples \
    -DartifactId=streams-flink-java-examples \
    -Dversion=0.1 \
    -Dpackage=com.github.masato.streams.flink \
    -DinteractiveMode=false
```

　プラグインの[maven-compiler-plugin](https://maven.apache.org/plugins/maven-compiler-plugin/)の設定をJava 8 (1.8)に変更します。また[exec-maven-plugin](http://www.mojohaus.org/exec-maven-plugin/)を追加してMavenからFlinkアプリの`main()`を実行できるようにします。


```xml pom.xml
    <build>
        <plugins>
...
              <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>3.1</version>
                <configuration>
                  <source>1.8</source>
                  <target>1.8</target>
                </configuration>
              </plugin>

              <plugin>
                <groupId>org.codehaus.mojo</groupId>
                <artifactId>exec-maven-plugin</artifactId>
                <version>1.5.0</version>
                <executions>
                  <execution>
                    <id>App</id>
                    <goals>
                      <goal>exec</goal>
                    </goals>
                  </execution>
                </executions>
                <configuration>
                  <executable>java</executable>
                  <classpathScope>compile</classpathScope>
                  <arguments>
                    <argument>-cp</argument>
                    <classpath/>
                    <argument>com.github.masato.streams.flink.WordCount</argument>
                  </arguments>
                </configuration>
              </plugin>
```

　[execゴール](http://www.mojohaus.org/exec-maven-plugin/examples/example-exec-for-java-programs.html)を実行します。[WordCount](https://github.com/apache/flink/blob/master/flink-quickstart/flink-quickstart-java/src/main/resources/archetype-resources/src/main/java/WordCount.java)の例はテキストの単語を数えて標準出力します。

```bash
$ mvn clean package exec:exec@App
...
(is,1)
(a,1)
(in,1)
(mind,1)
(or,2)
(against,1)
(arms,1)
(not,1)
(sea,1)
(the,3)
(troubles,1)
(fortune,1)
(take,1)
(to,4)
(and,1)
(arrows,1)
(be,2)
(nobler,1)
(of,2)
(slings,1)
(suffer,1)
(outrageous,1)
(tis,1)
(whether,1)
(question,1)
(that,1)
```

## ウィンドウ集計

　[flink-quickstart-java](https://mvnrepository.com/artifact/org.apache.flink/flink-quickstart-java)のアーキタイプが作成するJavaコードをすべて削除してから新しいコードを書いていきます。

```bash
$ rm src/main/java/com/github/masato/streams/flink/*.java
```
　
　ソースコードは[リポジトリ](https://github.com/masato/streams-flink-java-examples)にもあります。Scalaで書いた[例](https://github.com/masato/streams-flink-scala-examples/)は一つのファイルでしたがJavaの場合はわかりやすいようにクラスを分けました。

```bash
$ tree streams-flink-java-examples
streams-flink-java-examples
├── pom.xml
└── src
    └── main
        ├── java
        │   └── com
        │       └── github
        │           └── masato
        │               └── streams
        │                   └── flink
        │                       ├── Accumulator.java
        │                       ├── Aggregate.java
        │                       ├── App.java
        │                       ├── Average.java
        │                       └── Sensor.java
        └── resources
            └── log4j.properties
```

### App.java

 メインメソッドを実装したプログラムの全文です。Scalaで書いた例の[App.scala](https://github.com/masato/streams-flink-scala-examples/blob/master/src/main/scala/com/github/masato/streams/flink/App.scala)と似ていますが[AggregateFunction](https://ci.apache.org/projects/flink/flink-docs-release-1.3/api/java/org/apache/flink/api/common/functions/AggregateFunction.html)と[WindowFunction](https://ci.apache.org/projects/flink/flink-docs-release-1.3/api/java/org/apache/flink/streaming/api/functions/windowing/WindowFunction.html)はそれぞれクラスにしました。

```java App.java
package com.github.masato.streams.flink;

import java.util.Date;
import java.util.Map;
import java.util.HashMap;
import java.util.Properties;
import java.time.ZoneId;
import java.time.ZonedDateTime;
import java.time.format.DateTimeFormatter;

import org.apache.flink.streaming.api.TimeCharacteristic;
import org.apache.flink.streaming.api.datastream.DataStream;
import org.apache.flink.streaming.api.datastream.SingleOutputStreamOperator;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.streaming.api.functions.source.SourceFunction;
import org.apache.flink.streaming.api.functions.timestamps.BoundedOutOfOrdernessTimestampExtractor;
import org.apache.flink.streaming.api.windowing.time.Time;
import org.apache.flink.streaming.api.windowing.windows.TimeWindow;
import org.apache.flink.streaming.connectors.kafka.FlinkKafkaConsumer010;
import org.apache.flink.streaming.connectors.kafka.FlinkKafkaProducer010;
import org.apache.flink.streaming.util.serialization.JSONDeserializationSchema;
import org.apache.flink.streaming.util.serialization.SimpleStringSchema;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.node.ObjectNode;

public class App {
    private static DateTimeFormatter fmt = DateTimeFormatter.ISO_OFFSET_DATE_TIME;

    private static final String bootstrapServers = System.getenv("BOOTSTRAP_SERVERS_CONFIG");
    private static final String groupId = System.getenv("GROUP_ID");
    private static final String sourceTopic = System.getenv("SOURCE_TOPIC");

    private static final String sinkTopic = System.getenv("SINK_TOPIC");

    public static void main(String[] args) throws Exception {
        final Properties props = new Properties();
        props.setProperty("bootstrap.servers", bootstrapServers);
        props.setProperty("group.id", groupId);

        final StreamExecutionEnvironment env =
            StreamExecutionEnvironment.getExecutionEnvironment();
        env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime);

        final DataStream<ObjectNode> events =
                                env.addSource(new FlinkKafkaConsumer010<>(
                                  sourceTopic,
                                  new JSONDeserializationSchema(),
                                  props)).name("events");

        final SingleOutputStreamOperator<ObjectNode> timestamped =
            events
            .assignTimestampsAndWatermarks(
                new BoundedOutOfOrdernessTimestampExtractor<ObjectNode>(Time.seconds(10)) {
                    @Override
                    public long extractTimestamp(ObjectNode element) {
                        return element.get("time").asLong() * 1000;
                    }
                });

        timestamped
            .map((v) -> {
                    String key =  v.get("bid").asText();
                    double ambient = v.get("ambient").asDouble();
                    return new Sensor(key, ambient);
            })
            .keyBy(v -> v.key)
            .timeWindow(Time.seconds(60))
            .aggregate(new Aggregate(), new Average())
            .map((v) -> {
                    ZonedDateTime zdt =
                        new Date(v.time).toInstant().atZone(ZoneId.systemDefault());
                    String time = fmt.format(zdt);

                    Map<String, Object> payload = new HashMap<String, Object>();
                    payload.put("time", time);
                    payload.put("bid", v.bid);
                    payload.put("ambient", v.sum);

                    String retval = new ObjectMapper().writeValueAsString(payload);
                    System.out.println(retval);
                    return retval;
                })
            .addSink(new FlinkKafkaProducer010<String>(
                         bootstrapServers,
                         sinkTopic,
                         new SimpleStringSchema())
                     ).name("kafka");

        env.execute();
    }
}
```

### Sensor.java

　ScalaではストリームのセンサーデータはBDアドレスをキーにScalaのTupleを作成しました。

```scala
    timestamped
      .map { v =>
        val key =  v.get("bid").asText
        val ambient = v.get("ambient").asDouble
        (key, ambient)
      }
```

　Java 8の場合でもScalaのように[Tuple2](https://ci.apache.org/projects/flink/flink-docs-release-1.3/api/java/org/apache/flink/api/java/tuple/Tuple2.html)が使えます。しかし[Using Apache Flink with Java 8](https://brewing.codes/2017/01/31/using-apache-flink-with-java-8/)の解説にあるようにEclipse JDTでコンパイルが必要です。または[returns()](https://ci.apache.org/projects/flink/flink-docs-release-1.3/api/java/org/apache/flink/streaming/api/datastream/SingleOutputStreamOperator.html#returns-org.apache.flink.api.common.typeinfo.TypeInformation-)に[TupleTypeInfo](https://ci.apache.org/projects/flink/flink-docs-release-1.3/api/java/org/apache/flink/api/java/typeutils/TupleTypeInfo.html)で要素のタイプヒントをJavaクラスで指定しないとエラーになります。


```java
            .map((v) -> {
                double ambient = v.get("value").get("ambient").asDouble();
                String key =  v.get("sensor").get("bid").asText();
                return new Tuple2<>(key, ambient);
            })
            .returns(new TupleTypeInfo<>(TypeInformation.of(String.class),
                                         TypeInformation.of(Double.class)))

```

　ちょっと面倒なので普通のPOJOを利用したほうが簡単です。

```java Sensor.java
package com.github.masato.streams.flink;

public class Sensor {
    public String key;
    public double ambient;

    public Sensor(String key, double ambient) {
        this.key = key;
        this.ambient = ambient;
    }
}
```

### Aggregate.java

　[AggregateFunction](https://ci.apache.org/projects/flink/flink-docs-release-1.3/api/java/org/apache/flink/api/common/functions/AggregateFunction.html)インタフェースを実装します。Scalaと違いAccumulatorはcaseクラスではありませんがそれ以外はほぼ同じです。

```java Aggregate.java
package com.github.masato.streams.flink;

import org.apache.flink.api.java.tuple.Tuple2;

import org.apache.flink.api.common.functions.AggregateFunction;

public class Aggregate implements AggregateFunction<Sensor, Accumulator, Accumulator> {

    private static final long serialVersionUID = 3355966737412029618L;

    @Override
    public Accumulator createAccumulator() {
        return new Accumulator(0L, "", 0.0, 0);
    }

    @Override
    public Accumulator merge(Accumulator a, Accumulator b) {
        a.count += b.count;
        a.sum += b.sum;
        return a;
    }

    @Override
    public void add(Sensor value, Accumulator acc) {
        acc.sum += value.ambient;
        acc.count++;
    }

    @Override
    public Accumulator getResult(Accumulator acc) {
        return acc;
    }
}
```

### Average.java

　[WindowFunction](https://ci.apache.org/projects/flink/flink-docs-release-1.3/api/java/org/apache/flink/streaming/api/functions/windowing/WindowFunction.html)の実装です。

```java Average.java
package com.github.masato.streams.flink;

import org.apache.flink.streaming.api.windowing.windows.TimeWindow;
import org.apache.flink.streaming.api.functions.windowing.WindowFunction;
import org.apache.flink.util.Collector;

public class Average implements WindowFunction<Accumulator,
                                               Accumulator, String, TimeWindow> {

    private static final long serialVersionUID = 5532466889638450746L;

    @Override
    public void apply(String key,
                      TimeWindow window,
                      Iterable<Accumulator> input,
                      Collector<Accumulator> out) {

        Accumulator in = input.iterator().next();
        out.collect(new Accumulator(window.getEnd(), key, in.sum/in.count, in.count));
    }
}
```

　Scalaの場合WindowFunctionの`apply()`実装は`aggregate`には直接書いてみました。

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

### pom.xml

　ストリームのSourceはKafkaを利用します。接続情報は[exec-maven-plugin](http://www.mojohaus.org/exec-maven-plugin/)の環境変数に設定します。SensorTagとRaspberry Pi 3の準備、Kafkaクラスタの構築は[こちら](https://masato.github.io/2017/07/28/sensortag-kafka-python-spark-streaming-2/)を参考にしてください。

```xml pom.xml
    <build>
        <plugins>
...
              <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>3.1</version>
                <configuration>
                  <source>1.8</source>
                  <target>1.8</target>
                </configuration>
              </plugin>

              <plugin>
                <groupId>org.codehaus.mojo</groupId>
                <artifactId>exec-maven-plugin</artifactId>
                <version>1.5.0</version>
                <executions>
                  <execution>
                    <id>App</id>
                    <goals>
                      <goal>exec</goal>
                    </goals>
                  </execution>
                </executions>
                <configuration>
                  <executable>java</executable>
                  <classpathScope>compile</classpathScope>
                  <arguments>
                    <argument>-cp</argument>
                    <classpath/>
                    <argument>com.github.masato.streams.flink.App</argument>
                  </arguments>
                  <environmentVariables>
                    <APPLICATION_ID_CONFIG>sensortag</APPLICATION_ID_CONFIG>
                    <BOOTSTRAP_SERVERS_CONFIG>confluent:9092</BOOTSTRAP_SERVERS_CONFIG>
                    <SOURCE_TOPIC>sensortag</SOURCE_TOPIC>
                    <SINK_TOPIC>sensortag-sink</SINK_TOPIC>
                    <GROUP_ID>flinkGroup</GROUP_ID>
                  </environmentVariables>
                </configuration>
              </plugin>
```

### 実行

　Raspberry Pi 3からSensorTagのデータをKafkaに送信したあとにexec-maven-pluginのexecゴールを実行します。

```
$ mvn clean install exec:exec@App
```

　周囲温度(ambient)を60秒のタンブリングウィンドウで集計した平均値が標準出力されました。

```bash
{"ambient":28.395833333333332,"time":"2017-08-28T11:57:00+09:00","bid":"B0:B4:48:BD:DA:03"}
{"ambient":28.44375,"time":"2017-08-28T11:58:00+09:00","bid":"B0:B4:48:BD:DA:03"}
{"ambient":28.46875,"time":"2017-08-28T11:59:00+09:00","bid":"B0:B4:48:BD:DA:03"}
{"ambient":28.5,"time":"2017-08-28T12:00:00+09:00","bid":"B0:B4:48:BD:DA:03"}
```
