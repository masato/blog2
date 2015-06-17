title: "Spark on Dockerで分散型機械学習を始める - Part1: インストール"
date: 2015-01-10 16:37:57
tags:
 - Docker
 - Scala
 - Python
 - Spark
 - PySpark
 - SequenceIQ
 - MLib
 - Cloudbreak
description: 日経BPのITインフラテクノロジーAWARD 2015が発表されました。2015年にブレークすると予想されるクラウドやビッグデータの製品やサービスを選出しています。グランプリにDocker、準グランプリにApache Sparkが選ばれました。Sparkは2014年に入り盛り上がってきています。インメモリで高速に分散処理ができるため、機械学習のような繰り返し処理に向いています。MLibの機械学習ライブラリもあるので分散型機械学習フレームワークとして注目を集めています。そんなDockerとSparkを使い手軽に分散型機械学習の環境をつくり勉強していこうと思います。
---


日経BPの[ITインフラテクノロジーAWARD 2015](http://corporate.nikkeibp.co.jp/information/newsrelease/newsrelease20141224.shtml)が発表されました。2015年にブレークすると予想されるクラウドやビッグデータの製品やサービスを選出しています。グランプリにDocker、準グランプリにApache Sparkが選ばれました。Sparkは2014年に入り盛り上がってきています。インメモリで高速に分散処理ができるため、機械学習のような繰り返し処理に向いています。MLibの機械学習ライブラリもあるので分散型機械学習フレームワークとして注目を集めています。そんなDockerとSparkを使い手軽に分散型機械学習の環境をつくり勉強していこうと思います。

<!-- more -->


### Sparkの書籍

新しい技術を学習する場合、最初は書籍から網羅的にはいったほうが概念をつかみやすいです。

* [Advanced Analytics with Spark Patterns for Learning from Data at Scale](http://shop.oreilly.com/product/0636920035091.do)
* [Learning Spark Lightning-Fast Big Data Analytics](http://shop.oreilly.com/product/0636920028512.do)
* [Fast Data Processing with Spark](https://www.packtpub.com/big-data-and-business-intelligence/fast-data-processing-spark)

Advanced Analytics with Spark Patterns for Learning from Data at Scaleは[O’Reilly Web Ops & Performance Newsletter](http://radar.oreilly.com/webops-perf)からHappy Holidaysギフトでプレゼントしてもらいました。

### SequenceIQのDockerイメージ

Dockerイメージは[SequenceIQ](http://sequenceiq.com/)の[sequenceiq/spark](https://registry.hub.docker.com/u/sequenceiq/spark/)を使います。SequenceIQはHadoop-as-a-Service APIの[Cloudbreak](https://github.com/sequenceiq/cloudbreak)オープンソースで開発しています。CloudbreakはAmbariとDockerを使っているようです。

### インストール

[sequenceiq/spark](https://registry.hub.docker.com/u/sequenceiq/spark/)のDockerイメージをpullします。2014-12-18にリリースされたSpark 1.2.0を使います。

``` bash
$ docker pull sequenceiq/spark:1.2.0
```

コンテナを起動します。

``` bash
$ docker run -i -t -h sandbox sequenceiq/spark:1.2.0 /etc/bootstrap.sh -bash
/
Starting sshd:                                             [  OK  ]
Starting namenodes on [sandbox]
sandbox: starting namenode, logging to /usr/local/hadoop/logs/hadoop-root-namenode-sandbox.out
localhost: starting datanode, logging to /usr/local/hadoop/logs/hadoop-root-datanode-sandbox.out
Starting secondary namenodes [0.0.0.0]
0.0.0.0: starting secondarynamenode, logging to /usr/local/hadoop/logs/hadoop-root-secondarynamenode-sandbox.out
starting yarn daemons
starting resourcemanager, logging to /usr/local/hadoop/logs/yarn--resourcemanager-sandbox.out
localhost: starting nodemanager, logging to /usr/local/hadoop/logs/yarn-root-nodemanager-sandbox.out
bash-4.1#
```

最後の`-bash`は[/etc/bootstrap.sh](https://github.com/sequenceiq/hadoop-docker/blob/master/bootstrap.sh
)を実行するときのフラグです。`/bin/bash`を起動します。

``` bash bootstrap.sh
...
if [[ $1 == "-d" ]]; then
  while true; do sleep 1000; done
fi

if [[ $1 == "-bash" ]]; then
  /bin/bash
fi
```

### インストールとバーションの確認

Hadoopのバージョンは2.6.0です。

``` bash
$ hadoop version
Hadoop 2.6.0
Subversion https://git-wip-us.apache.org/repos/asf/hadoop.git -r e3496499ecb8d220fba99dc5ed4c99c8f9e33bb1
Compiled by jenkins on 2014-11-13T21:10Z
Compiled with protoc 2.5.0
From source with checksum 18e43357c8f927c0695f1e9522859d6a
This command was run using /usr/local/hadoop-2.6.0/share/hadoop/common/hadoop-common-2.6.0.jar
```

Sparkのバージョンは1.2.0、Scalaのバージョンは2.10.4です。

``` bash
$ spark-shell
...
Welcome to
      ____              __
     / __/__  ___ _____/ /__
    _\ \/ _ \/ _ `/ __/  '_/
   /___/ .__/\_,_/_/ /_/\_\   version 1.2.0
      /_/

Using Scala version 2.10.4 (Java HotSpot(TM) 64-Bit Server VM, Java 1.7.0_51)
...
scala> :quit
```

Sparkのインストールディレクトリです。

``` bash
$ echo $SPARK_HOME
$ /usr/local/spark
```

Hadoopのインストールディレクトリです。

``` bash
$ echo HADOOP_YARN_HOME
/usr/local/hadoop
```

spark-shellとpysparkコマンドは`$SPARK_HOME/bin`に配置されています。

``` bash
$ which spark-shell
/usr/local/spark/bin/spark-shell
$ which pyspark
/usr/local/spark/bin/pyspark
```

### Spark Shell (ScalaとPython)

SequenceIQのブログ[Apache Spark 1.2.0 on Docker](http://blog.sequenceiq.com/blog/2015/01/09/spark-1-2-0-docker/)を読みながら試してみます。[Quick Start](https://spark.apache.org/docs/1.2.0/quick-start.html)にもSpark Shellのサンプルがあります。

インタラクティブ分析に使うSpark Shellには、Scalaのspark-shellとPythonのpysparkが用意されています。はじめにScala APIのspark-shellを起動してサンプルコードを実行します。

``` bash
$ spark-shell
...
Welcome to
      ____              __
     / __/__  ___ _____/ /__
    _\ \/ _ \/ _ `/ __/  '_/
   /___/ .__/\_,_/_/ /_/\_\   version 1.2.0
      /_/

Using Scala version 2.10.4 (Java HotSpot(TM) 64-Bit Server VM, Java 1.7.0_51)
Type in expressions to have them evaluated.
...
scala> sc.parallelize(1 to 1000).count()
...
res0: Long = 1000
```

次にSparkのPython APIであるPySparkを使います。pysparkを起動して同様のサンプルコードを実行します。

``` bash
$ pyspark
...
Welcome to
      ____              __
     / __/__  ___ _____/ /__
    _\ \/ _ \/ _ `/ __/  '_/
   /__ / .__/\_,_/_/ /_/\_\   version 1.2.0
      /_/

Using Python version 2.6.6 (r266:84292, Jan 22 2014 09:42:36)
SparkContext available as sc.
>>> sc.parallelize(range(1000)).count()
...
1000
``` 


### 円周率計算のサンプル

spark-examples-1.2.0-hadoop2.4.0.jarから円周率計算のサンプルプログラムを実行します。

YARN上で実行するSparkアプリは2つのモードがあります。

* yarn-cluster mode
 * SparkアプリはYARNクラスタで実行される
 * 通常のバッチ処理で使う
* yarn-client mode
 * Sparkアプリはローカルホストで実行される
 * デバッグなどインタラクティブ処理で使う

yarn-cluster modeで実行すると、処理結果は`$HADOOP_YARN_HOME/logs`に出力されます。

``` bash
$ spark-submit --class org.apache.spark.examples.SparkPi --master yarn-cluster --driver-memory 1g --executor-memory 1g --executor-cores 1 $SPARK_HOME/lib/spark-examples-1.2.0-hadoop2.4.0.jar
...
15/01/10 02:00:43 INFO yarn.Client:
         client token: N/A
         diagnostics: N/A
         ApplicationMaster host: sandbox
         ApplicationMaster RPC port: 0
         queue: default
         start time: 1420873225740
         final status: SUCCEEDED
         tracking URL: http://sandbox:8088/proxy/application_1420873088326_0001/A
         user: root
```

ログを確認します。

``` bash
$ cat /usr/local/hadoop/logs/userlogs/application_1420873088326_0001/container_1420873088326_0001_01_000001/stdout
Pi is roughly 3.1451
```

yarn-client modeで実行すると、処理結果はコンソールに標準出力されます。

``` bash
$ spark-submit --class org.apache.spark.examples.SparkPi --master yarn-client --driver-memory 1g --executor-memory 1g --executor-cores 1 $SPARK_HOME/lib/spark-examples-1.2.0-hadoop2.4.0.jar
...
Pi is roughly 3.14515
```
