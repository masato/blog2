title: "Cassandra on DockerでIoT用データストアを用意する - Part1: Single Nodeインストール"
date: 2015-01-19 16:10:23
tags:
 - Cassandra
 - IoT
 - Spotify
 - KairosDB
 - Blueflood
 - Docker
 - 時系列データベース
description: IoT用のデータストアとして使える時系列データベースを調査しています。オープンソースのBluefloodとKairosDBや、商用IoTプラットフォームのThingWorxのバックエンドにCassandraが採用されています。
---

IoT用のデータストアとして使える時系列データベースを調査しています。オープンソースの[Blueflood](http://blueflood.io/)と[KairosDB](https://github.com/kairosdb/kairosdb)や、商用IoTプラットフォームの[ThingWorx](http://www.thingworx.com/)のバックエンドにCassandraが採用されています。

<!-- more -->

## IoTに適した時系列データベース

IoT用の時系列データベースはバックエンドにCassandraやMongoDB、HBaseを使うケースが多いようです。商用だと[Realtime.co](http://www.realtime.co/)のBaaSは[DynamoDB](http://aws.amazon.com/jp/dynamodb/)を、[TempoIQ](https://www.tempoiq.com/)は以前TempoDBと呼ばれていましたが、社名を変えエンタープライズ向けのセンサーデータ解析に特化したサービスになりました。

* [Blueflood](http://blueflood.io/)
* [KairosDB](https://github.com/kairosdb/kairosdb)
* [InfluxDB](http://influxdb.com/)
* [OpenTSDB](http://opentsdb.net/)
* [Druid](http://druid.io/)
* [TempoIQ](https://www.tempoiq.com/)
* [ThingWorx](http://www.thingworx.com/)

## spotify/docker-cassandra

IoT用のデータストアにはCassandraベースだと相性が良さそうな気がするので、DockerにCassandraの学習環境を用意しようと思います。[Docker Registry Hub](https://registry.hub.docker.com/)で良さそうなDockerイメージを探します。

* [spotify/docker-cassandra](https://registry.hub.docker.com/u/spotify/cassandra/)
* [poklet/cassandra](https://registry.hub.docker.com/u/poklet/cassandra/)
* [abh1nav/docker-cassandra](https://registry.hub.docker.com/u/abh1nav/cassandra/)

いくつか見つかりましたが、[Luigi](https://github.com/spotify/luigi)や[Snakebite](https://github.com/spotify/snakebite)が気に入っているので、[Spotify](https://developer.spotify.com/)のDockerイメージを使ってみます。SpotifyではプレイリストのパーソナライゼーションにCassandraを使っています。

* [Personalization at Spotify using Cassandra](https://labs.spotify.com/2015/01/09/personalization-at-spotify-using-cassandra/)
* [Spotify scales to the top of the charts with Apache Cassandra at 40k requests/second](http://planetcassandra.org/blog/interview/spotify-scales-to-the-top-of-the-charts-with-apache-cassandra-at-40k-requestssecond/)

## CoreOSホスト

CoreOSクラスタの1台にログインします。しばらくログインしないでいるとバージョンが`522.4.0`に上がっていました。

``` bash
$ cat /etc/os-release
NAME=CoreOS
ID=coreos
VERSION=522.4.0
VERSION_ID=522.4.0
BUILD_ID=
PRETTY_NAME="CoreOS 522.4.0"
ANSI_COLOR="1;32"
HOME_URL="https://coreos.com/"
BUG_REPORT_URL="https://github.com/coreos/bugs/issues"
```

Dockerのバージョンは`1.3.3`です。

``` bash
$ docker -version
docker version
Client version: 1.3.3
Client API version: 1.15
Go version (client): go1.3.2
Git commit (client): 54d900a
OS/Arch (client): linux/amd64
Server version: 1.3.3
Server API version: 1.15
Go version (server): go1.3.2
Git commit (server): 54d900a
```

## Single Node起動

[spotify/docker-cassandra](https://registry.hub.docker.com/u/spotify/cassandra/)のDockerイメージを使います。最初のSingle Node起動なので、fleetのunitファイルは作成せずに直接`docker run`で起動します。

``` bash
$ docker run -d  --name cassandra spotify/cassandra
```

`docker ps`で起動を確認します。

``` bash
$ docker ps|head -2
CONTAINER ID        IMAGE                                       COMMAND                CREATED             STATUS              PORTS                                                                           NAMES
f955a0d522ce        spotify/cassandra:latest                    "cassandra-singlenod   44 seconds ago      Up 44 seconds       7001/tcp, 7199/tcp, 8012/tcp, 9042/tcp, 9160/tcp, 22/tcp, 61621/tcp, 7000/tcp   cassandra                                                                                 cassandra
```

`docker exec`からコンテナのbashを起動して、Cassandraの動作確認をします。

``` bash
$ docker exec -it cassandra /bin/bash
```

## cassandra-cliの確認

Cassandraクライアントのcassandra-cliを起動します。Cassandraのバージョンは`2.0.10`です。

``` bash
$ cassandra-cli
Connected to: "Test Cluster" on 127.0.0.1/9160
Welcome to Cassandra CLI version 2.0.10

The CLI is deprecated and will be removed in Cassandra 3.0.  Consider migrating to cqlsh.
CQL is fully backwards compatible with Thrift data; see http://www.datastax.com/dev/blog/thrift-to-cql3

Type 'help;' or '?' for help.
Type 'quit;' or 'exit;' to quit.

[default@unknown]
```

RDBのデータベース相当のキースペースを確認します。

``` bash
[default@unknown] show keyspaces;
...
Keyspace: system_traces:
  Replication Strategy: org.apache.cassandra.locator.SimpleStrategy
  Durable Writes: true
    Options: [replication_factor:2]
  Column Families:
```

## cqlshの確認

SQLライクに問い合わせができるCQLの確認をします。
 
``` bash
$ cqlsh
Connected to Test Cluster at localhost:9160.
[cqlsh 4.1.1 | Cassandra 2.0.10 | CQL spec 3.1.1 | Thrift protocol 19.39.0]
Use HELP for help.
cqlsh>
```

キースペースを確認します。

``` bash
cqlsh> DESCRIBE keyspaces;

system  system_traces
```

CQLには`CREATE TABLE`で定義できるテーブルがあります。

``` bash
cqlsh> DESCRIBE tables;

Keyspace system
---------------
IndexInfo                hints        range_xfers            sstable_activity
NodeIdInfo               local        schema_columnfamilies
batchlog                 paxos        schema_columns
compaction_history       peer_events  schema_keyspaces
compactions_in_progress  peers        schema_triggers

Keyspace system_traces
----------------------
events  sessions
```

