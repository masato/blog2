title: "PythonとSensorTag, Kafka, Spark Streamingのストリーム処理 - Part 5: Apache Toree でJupyterからSparkに接続する"
date: 2017-08-01 09:05:19
categories:
 - IoT
tags:
 - Spark
 - Python
 - Scala
description: parkクラスタを用意していくつかサンプルコードを書いていこうと思います。Pythonのデータ分析や機械学習の実行環境としてJupyterは多くの方が利用していると思います。Apache ToreeでSparkアプリも同じようにJupyterからインタラクティブに書くことが目的です。ブラウザから実行できるScalaのREPLしてもJupyterを使うことができます。
---

　Sparkクラスタを用意していくつかサンプルコードを書いていこうと思います。Pythonのデータ分析や機械学習の実行環境としてJupyterは多くの方が利用していると思います。[Apache Toree](https://toree.apache.org/)でSparkアプリも同じようにJupyterからインタラクティブに書くことが目的です。ブラウザから実行できるScalaのREPLしてもJupyterを使うことができます。

<!-- more -->

## Spark

　SparkクラスタをDocker Composeで構築します。Docker HubとGitHubに多くのSpark Standalone Cluster用のイメージとdocker-compose.ymlが公開されています。

 * [semantive/spark](https://github.com/Semantive/docker-spark)
 * [produktion/jupyter-pyspark](https://github.com/maltefiala/docker--jupyter-pyspark)
 * [gettyimages/docker-spark](https://github.com/gettyimages/docker-spark/)

　いくつか試しましたが[semantive/spark](https://github.com/Semantive/docker-spark)がシンプルで使いやすい印象です。

### Docker Compose

　`semantive/spark`イメージの使い方は[Docker Images For Apache Spark](http://semantive.com/docker-images-for-apache-spark/)に書いてあります。Docker Hubは[こちら](https://hub.docker.com/r/semantive/spark/)、GitHubは[こちら](https://github.com/Semantive/docker-spark)になります。

　リポジトリにある[docker-compose.yml](https://github.com/Semantive/docker-spark/blob/master/docker-compose.yml)からいくつか変更しました。主な変更点はSparkのバージョンを合わせるためイメージタグを明示的に指定する、`SPARK_PUBLIC_DNS`と`SPARK_MASTER_HOST`環境変数にクラウド上の仮想マシンのパブリックIPアドレスを指定することです。

```yaml docker-compose.yml
version: '2'
services:
  master:
    image: semantive/spark:spark-2.1.1-hadoop-2.7.3
    command: bin/spark-class org.apache.spark.deploy.master.Master -h master
    hostname: master
    environment:
      MASTER: spark://master:7077
      SPARK_CONF_DIR: /conf
      SPARK_PUBLIC_DNS: <仮想マシンのパブリックIPアドレス>
      SPARK_MASTER_HOST: <仮想マシンのパブリックIPアドレス>
    ports:
      - 4040:4040
      - 6066:6066
      - 7077:7077
      - 8080:8080
    volumes:
      - spark_data:/tmp/data

  worker1:
    image: semantive/spark:spark-2.1.1-hadoop-2.7.3
    command: bin/spark-class org.apache.spark.deploy.worker.Worker spark://master:7077
    hostname: worker1
    environment:
      SPARK_CONF_DIR: /conf
      SPARK_WORKER_CORES: 4
      SPARK_WORKER_MEMORY: 2g
      SPARK_WORKER_PORT: 8881
      SPARK_WORKER_WEBUI_PORT: 8081
      SPARK_PUBLIC_DNS: <仮想マシンのパブリックIPアドレス>
    depends_on:
      - master
    ports:
      - 8081:8081
    volumes:
      - spark_data:/tmp/data

  worker2:
    image: semantive/spark:spark-2.1.1-hadoop-2.7.3
    command: bin/spark-class org.apache.spark.deploy.worker.Worker spark://master:7077
    hostname: worker2
    environment:
      SPARK_CONF_DIR: /conf
      SPARK_WORKER_CORES: 4
      SPARK_WORKER_MEMORY: 2g
      SPARK_WORKER_PORT: 8882
      SPARK_WORKER_WEBUI_PORT: 8082
      SPARK_PUBLIC_DNS: <仮想マシンのパブリックIPアドレス>
    depends_on:
      - master
    ports:
      - 8082:8082
    volumes:
      - spark_data:/tmp/data

volumes:
  spark_data:
    driver: local
```

　Spark Standalone Clusterを起動します。

```
$ docker-compose up -d
```

　Spark Master UIを開いてクラスタの状態を確認します。

![spark-standalone.png](/2017/08/01/sensortag-kafka-python-spark-streaming-5/spark-standalone.png)

　Masterコンテナのspark-shellを実行してScalaとSparkのバージョンを確認します。Sparkは開発のスピードがとても速く、Scalaのバージョンも含めてよく確認しないと思わぬエラーに遭遇してしまいます。
　
* Scala: 2.11.8
* Spark: 2.1.1

```
$ docker-compose exec master spark-shell
...
Welcome to
      ____              __
     / __/__  ___ _____/ /__
    _\ \/ _ \/ _ `/ __/  '_/
   /___/ .__/\_,_/_/ /_/\_\   version 2.1.1
      /_/

Using Scala version 2.11.8 (OpenJDK 64-Bit Server VM, Java 1.8.0_131)
Type in expressions to have them evaluated.
Type :help for more information.

scala>
```

## Jupyter

　JupyterのDockerイメージは公式の[jupyter/all-spark-notebook](https://hub.docker.com/r/jupyter/all-spark-notebook/)を使います。ScalaやSparkまで使える全部入りのイメージです。
　
### Apache Toree
　
　[Apache Toree](https://toree.apache.org/)はSparkクラスタにJupyterから接続するためのツールです。PySparkに加え、Scala、SparkR、SQLのKernelが提供されます。

　[Dockerfile](https://github.com/jupyter/docker-stacks/blob/master/all-spark-notebook/Dockerfile)を見るとApache Toreeもインストールされています。


```Dockerfile
# Apache Toree kernel
RUN pip --no-cache-dir install https://dist.apache.org/repos/dist/dev/incubator/toree/0.2.0/snapshots/dev1/toree-pip/toree-0.2.0.dev1.tar.gz
RUN jupyter toree install --sys-prefix
```

### docker-compose.yml

　Spark Standalone Clusterのdocker-compose.ymlにJupyterサービスを追加します。
　
```yaml docker-compose.yml
  jupyter:
    image: jupyter/all-spark-notebook:c1b0cf6bf4d6
    depends_on:
      - master
    ports:
      - 8888:8888
    volumes:
      - ./notebooks:/home/jovyan/work
      - ./ivy2:/home/jovyan/.ivy2
    env_file:
      - ./.env
    environment:
      TINI_SUBREAPER: 'true'
      SPARK_OPTS: --master spark://master:7077 --deploy-mode client --packages com.amazonaws:aws-java-sdk:1.7.4,org.apache.hadoop:hadoop-aws:2.7.3
    command: start-notebook.sh --NotebookApp.password=sha1:xxx --NotebookApp.iopub_data_rate_limit=10000000
```

## Jupyterサービスのオプションについて

　Spark Standalone ClusterではHadoopを利用していないため分散ファイルシステムにAmazon S3を利用する設定を追加しています。サンプルデータやParquetファイルの保存先にあると便利です。

### image

　`jupyter/all-spark-notebook`イメージは更新が頻繁に入ります。Apache Toreeで使うSparkとSparkクラスタのバージョンがエラーになり起動しなくなります。今回はSparkクラスタのバージョンは`2.1.1`なので同じバージョンのイメージのtagを指定します。`jupyter/all-spark-notebook`イメージのタグはIDしかわからないのが不便です。

　Sparkのバージョンはすでに[2.2.0](https://github.com/jupyter/docker-stacks/commit/c740fbb1ca63db5856e004d29dd08d11fb4f91f8)へ上がっているため、`2.1.1`のタグを指定します。
　
　タグのDockerイメージをpullして`spark-shell`で確認します。

```
$ docker pull jupyter/all-spark-notebook:c1b0cf6bf4d6
$ docker run -it --rm \
  jupyter/all-spark-notebook:c1b0cf6bf4d6 \
  /usr/local/spark-2.1.1-bin-hadoop2.7/bin/spark-shell
```

　SparkクラスタとSparkとScalaのバージョンが同じであることが確認できました。

```
Welcome to
      ____              __
     / __/__  ___ _____/ /__
    _\ \/ _ \/ _ `/ __/  '_/
   /___/ .__/\_,_/_/ /_/\_\   version 2.1.1
      /_/

Using Scala version 2.11.8 (OpenJDK 64-Bit Server VM, Java 1.8.0_131)
Type in expressions to have them evaluated.
Type :help for more information.

scala>
```

　Jupyterのバージョンも確認しておきます。

```
$ docker run -it --rm jupyter/all-spark-notebook:c1b0cf6bf4d6 jupyter --version
4.3.0
```

### TINI_SUBREAPERとSPARK_OPTS

　Apache Toreeを利用してJupyterからリモートのSparkに接続するために必須な設定はこの2つです。`TINI_SUBREAPER`環境変数はinitに[Tini](https://github.com/krallin/tini)を使います。

　Sparkで追加のJarファイルを使わない場合は`SPARK_OPTS`環境変数に以下の指定だけでリモートのSpark Standalone Clusterに接続できます。通常のspark-submitのオプションと同じです。

```
--master spark://master:7077 --deploy-mode client
```

　追加のJarファイルがある場合はさらに`--packages`フラグを追加します。この場合はAmazon S3に接続するために必要なパッケージです。

```
--packages com.amazonaws:aws-java-sdk:1.7.4,org.apache.hadoop:hadoop-aws:2.7.3
```

### --NotebookApp.iopub_data_rate_limit

　[Bokeh](http://bokeh.pydata.org/en/latest/)など可視化ツールで大きな画像イメージを扱う場合はJupyterの起動スクリプトのオプションを指定します。

* 参考

[IOPub data rate exceeded when viewing image in Jupyter notebook](https://stackoverflow.com/questions/43288550/iopub-data-rate-exceeded-when-viewing-image-in-jupyter-notebook)

#### --NotebookApp.password

　Jupyterの認証方法はデフォルトはtokenです。Dockerコンテナのように頻繁に起動と破棄を繰り返す場合に毎回異なるtokeを入れるのは面倒なのでパスワード認証に変更しました。ipythonを使いパスワードのハッシュ値を取得します。
　
```python
$ docker run -it --rm jupyter/all-spark-notebook:c1b0cf6bf4d6 ipython
Python 3.6.1 | packaged by conda-forge | (default, May 23 2017, 14:16:20)
Type 'copyright', 'credits' or 'license' for more information
IPython 6.1.0 -- An enhanced Interactive Python. Type '?' for help.
```

　パスワードは以下のように生成します。出力されたハッシュ値をJupyterの起動オプションに指定します。

```
In [1]: from notebook.auth import passwd
In [2]: passwd()

Enter password:
Verify password:
Out[2]: 'sha1:xxx'
```

### volumes

　`/home/jovyan`はJupyterコンテナを実行しているユーザーのホームディレクトリです。作成したnotebookやダンロードしたJarファイルをDockerホストにマウントします。


### env_file

　`.env`ファイルに環境変数を記述してコンテナに渡します。Amazon S3への接続に使うaccess key と secret keyを指定します。

```
AWS_ACCESS_KEY_ID=xxx
AWS_SECRET_ACCESS_KEY=xxx
```

　Gitにcommitしないように忘れずに.gitignoreにも追加します。

```
.env
```

## JupyterからSparkとAmazon S3を使う

　JupyterでSparkとAmazon S3を使うサンプルをScalaとPythonで書いてみようと思います。[Monitoring Real-Time Uber Data Using Apache APIs, Part 1: Spark Machine Learning](https://dzone.com/articles/monitoring-real-time-uber-data-using-apache-apis-p)の記事で利用しているUberのピックアップデータをサンプルに使います。ここでは単純にCSVファイルをS3から読み込んで表示するだけです。

　docker-compose.ymlに定義した全てのサービスを起動します。
　
```
$ docker-compose up -d
```

### データ準備

　リポジトリをcloneしたあと`uber.csv`ファイルを`s3cmd`から適当なバケットにputします。

```
$ git clone https://github.com/caroljmcdonald/spark-ml-kmeans-uber
$ cd spark-ml-kmeans-uber/data
$ s3cmd put uber.csv s3://<バケット名>/uber-csv/
```

### Scala

　以下のようなコードを確認したいところでセルに分割してインタラクティブに実行することができます。ScalaのNotebookを書く場合は右上の`New`ボタンから`Apache Toree - Scala`を選択します。
　

```scala
import org.apache.spark.sql.SparkSession

val spark = SparkSession.
    builder.
    getOrCreate()

sc.hadoopConfiguration.set("fs.s3a.impl", "org.apache.hadoop.fs.s3a.S3AFileSystem")
sc.hadoopConfiguration.set("fs.s3a.fast.upload", "true")

import org.apache.spark.sql.types._
import org.apache.spark.sql.functions._

val schema = StructType(
    StructField("dt", TimestampType, true) ::
    StructField("lat", DoubleType, true) ::
    StructField("lon", DoubleType, true) ::
    StructField("base", StringType, true) :: Nil
)

val df = 
    spark.read.
    option("header", false).
    schema(schema).
    csv("s3a://<バケット名>/uber-csv/uber.csv")

df.printSchema

df.cache
df.show(false)
```

　Scalaの場合スキーマのStructTypeは次のようにも書くことができます。

```scala
val schema = (new StructType).
    add("dt", "timestamp", true).
    add("lat", "double", true).
    add("lon", "double", true).
    add("base", "string", true)
```


　最後の`df.show(false)`の出力結果です。
　
```
+---------------------+-------+--------+------+
|dt                   |lat    |lon     |base  |
+---------------------+-------+--------+------+
|2014-08-01 00:00:00.0|40.729 |-73.9422|B02598|
|2014-08-01 00:00:00.0|40.7476|-73.9871|B02598|
|2014-08-01 00:00:00.0|40.7424|-74.0044|B02598|
|2014-08-01 00:00:00.0|40.751 |-73.9869|B02598|
|2014-08-01 00:00:00.0|40.7406|-73.9902|B02598|
|2014-08-01 00:00:00.0|40.6994|-73.9591|B02617|
|2014-08-01 00:00:00.0|40.6917|-73.9398|B02617|
|2014-08-01 00:00:00.0|40.7063|-73.9223|B02617|
|2014-08-01 00:00:00.0|40.6759|-74.0168|B02617|
|2014-08-01 00:00:00.0|40.7617|-73.9847|B02617|
|2014-08-01 00:00:00.0|40.6969|-73.9064|B02617|
|2014-08-01 00:00:00.0|40.7623|-73.9751|B02617|
|2014-08-01 00:00:00.0|40.6982|-73.9669|B02617|
|2014-08-01 00:00:00.0|40.7553|-73.9253|B02617|
|2014-08-01 00:00:00.0|40.7325|-73.9876|B02682|
|2014-08-01 00:00:00.0|40.6754|-74.017 |B02682|
|2014-08-01 00:00:00.0|40.7303|-74.0029|B02682|
|2014-08-01 00:00:00.0|40.7218|-73.9973|B02682|
|2014-08-01 00:00:00.0|40.7134|-74.0091|B02682|
|2014-08-01 00:00:00.0|40.7194|-73.9964|B02682|
+---------------------+-------+--------+------+
only showing top 20 rows
```

### Python

　Python 3のNotebookを書く場合は右上の`New`ボタンから`Python 3`を選択します。以下のコードを適当なところでセルに分割して実行していきます。Scalaと異なるのは追加Jarは`PYSPARK_SUBMIT_ARGS`環境変数に指定する点です。

　以下のようにPythonでもほぼScalaと同じようにでSparkアプリを書くことができます。

``` python
import os
os.environ['PYSPARK_SUBMIT_ARGS'] = '--packages com.amazonaws:aws-java-sdk:1.7.4,org.apache.hadoop:hadoop-aws:2.7.3 pyspark-shell'

from pyspark.sql import SparkSession

spark = (
    SparkSession.builder
        .getOrCreate()
)

sc = spark.sparkContext

sc._jsc.hadoopConfiguration().set("fs.s3a.impl", "org.apache.hadoop.fs.s3a.S3AFileSystem")
sc._jsc.hadoopConfiguration().set("fs.s3a.fast.upload", "true")

from pyspark.sql.types import *
from pyspark.sql.functions import *

schema = StructType([
    StructField("dt", TimestampType(), True),
    StructField("lat", DoubleType(), True),
    StructField("lon", DoubleType(), True),
    StructField("base", StringType(), True)
])

df = (
    spark.read
    .option("header", False)
    .schema(schema)
    .csv("s3a://<バケット名>/uber-csv/uber.csv")
)

df.printSchema()

df.cache()
df.show(truncate=False)
```

　最後の`df.show(truncate=False)`の出力結果は先ほどのScalaのコードと同じです。

```
+---------------------+-------+--------+------+
|dt                   |lat    |lon     |base  |
+---------------------+-------+--------+------+
|2014-08-01 00:00:00.0|40.729 |-73.9422|B02598|
|2014-08-01 00:00:00.0|40.7476|-73.9871|B02598|
|2014-08-01 00:00:00.0|40.7424|-74.0044|B02598|
|2014-08-01 00:00:00.0|40.751 |-73.9869|B02598|
|2014-08-01 00:00:00.0|40.7406|-73.9902|B02598|
|2014-08-01 00:00:00.0|40.6994|-73.9591|B02617|
|2014-08-01 00:00:00.0|40.6917|-73.9398|B02617|
|2014-08-01 00:00:00.0|40.7063|-73.9223|B02617|
|2014-08-01 00:00:00.0|40.6759|-74.0168|B02617|
|2014-08-01 00:00:00.0|40.7617|-73.9847|B02617|
|2014-08-01 00:00:00.0|40.6969|-73.9064|B02617|
|2014-08-01 00:00:00.0|40.7623|-73.9751|B02617|
|2014-08-01 00:00:00.0|40.6982|-73.9669|B02617|
|2014-08-01 00:00:00.0|40.7553|-73.9253|B02617|
|2014-08-01 00:00:00.0|40.7325|-73.9876|B02682|
|2014-08-01 00:00:00.0|40.6754|-74.017 |B02682|
|2014-08-01 00:00:00.0|40.7303|-74.0029|B02682|
|2014-08-01 00:00:00.0|40.7218|-73.9973|B02682|
|2014-08-01 00:00:00.0|40.7134|-74.0091|B02682|
|2014-08-01 00:00:00.0|40.7194|-73.9964|B02682|
+---------------------+-------+--------+------+
only showing top 20 rows
```