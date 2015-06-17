title: "DockerのInfluxDBとGrafanaでdweet.ioのデータを可視化する"
date: 2015-02-11 01:06:15
tags:
 - InfluxDB
 - Gfarafa
 - Elasticsearch
 - dweetio
 - Docker
 - Tutum
description: Tutumが提供しているGrafanaとElasticsearchとGrafanaのDockerイメージを使ってIoT用のデータストアと可視化プラットフォームを構築します。MQTTブローカーのMoscaとまだ連携していないので、ダミーデータとしてdweet.ioのデータをInfluxDBクライアント使いデータベースに取り込みます。dweet.io + InfluxDB + Grafanaの投稿を参考に、カリフォルニアのアボガド果樹園のセンサーデータをdweet.ioから取得します。
---

[Tutum](https://www.tutum.co/)が提供しているGrafanaとElasticsearchとGrafanaのDockerイメージを使ってIoT用のデータストアと可視化プラットフォームを構築します。MQTTブローカーのMoscaとまだ連携していないので、ダミーデータとして[dweet.io](http://dweet.io/)のデータをInfluxDBクライアント使いデータベースに取り込みます。[dweet.io + InfluxDB + Grafana](http://datadventures.ghost.io/2014/09/07/dweet-io-influxdb-grafana/)の投稿を参考に、カリフォルニアのアボガド果樹園のセンサーデータをdweet.ioから取得します。

<!-- more -->

## InfluxDBコンテナ

IoTのデータストアとしてInfluxDBを使います。Dockerイメージは[tutum/influxdb](https://registry.hub.docker.com/u/tutum/influxdb/)です。

``` bash
$ docker pull tutum/influxdb
$ docker run --name influxdb \
  -d \
  -p 8083:8083 \
  -p 8086:8086 \
  --expose 8090 \
  --expose 8099 \
  -e PRE_CREATE_DB="influxdb"  \
  tutum/influxdb
```

ブラウザで確認します。usernameとpasswordは以下です。

* uesrname: root
* password root

http://10.1.3.67:8083

![influxdb-login.png](/2015/02/11/influxdb-grafana-dweet-example-on-docker/influxdb-login.png)

curlでレコードの登録と取得のテストをします。

``` bash
$ curl -X POST -d '[{"name":"foo","columns":["val"],"points":[[23]]}]' 'http://localhost:8086/db/influxdb/series?u=root&p=root'
```

レコードをクエリしてみます。timeとsequence_numberは自動的に入ります。

``` bash
$ curl -G 'http://localhost:8086/db/influxdb/series?u=root&p=root&pretty=true' --data-urlencode "q=select * from foo"
[
    {
        "name": "foo",
        "columns": [
            "time",
            "sequence_number",
            "val"
        ],
        "points": [
            [
                1423550210688,
                10001,
                23
            ]
        ]
    }
]
```

## Elasticsearchコンテナ

ElasticsearchにはGrafanaのダッシュボード設定情報を保存します。Dockerベースイメージは[tutum/elasticsearch](https://registry.hub.docker.com/u/tutum/elasticsearch/)です。今回のDockerホストの環境はIPv6が無効になっているため、Docker Hub Registryから取得したイメージは起動に失敗してしまいます。IPv6を無効にするためNginxの設定ファイルを修正してDockerイメージは作り直します。

``` bash
$ mkdir -p ~/docker_apps/es
$ cd !$
```

sedでIPv6の設定をコメントアウトして、Dockerfileを作成します。

``` bash Dockerfile
FROM tutum/elasticsearch
RUN sed -i  '/listen \[::\]:9200/s/\(listen \[::\]:9200.*\)/#\1/' /etc/nginx/sites-enabled/default
```

イメージをビルドしてコンテナを起動します。

``` bash
$ docker build masato/elasticsearch .
$ docker run \
  --name es \
  -d \
  -p 9200:9200 \
  -e ELASTICSEARCH_USER=admin \
  -e ELASTICSEARCH_PASS=mypass \
  masato/elasticsearch
```

curlでElasticsearchの起動を確認します。

``` bash
$ curl admin:mypass@10.1.3.67:9200
  "status" : 200,
  "name" : "Madcap",
  "version" : {
    "number" : "1.3.2",
    "build_hash" : "dee175dbe2f254f3f26992f5d7591939aaefd12f",
    "build_timestamp" : "2014-08-13T14:29:30Z",
    "build_snapshot" : false,
    "lucene_version" : "4.9"
  },
  "tagline" : "You Know, for Search"
}
```


## Grafanaコンテナ

GrafanaのDockerベースイメージは[tutum/grafana](https://registry.hub.docker.com/u/tutum/grafana/)です。Elasticsearchと同様にNginxのIPv6の設定をコメントアウトします。Dockerイメージをビルドするプロジェクトのディレクトリを作成します。

``` bash
$ mkdir -p ~/docker_apps/grafana
$ cd !$
```

Dockerfileを作成します。sedでNginxのIPv6設定をコメントアウトします。

``` bash Dockerfile
FROM tutum/grafana
RUN sed -i '/listen \[::\]:80/s/\(listen \[::\]:80.*\)/#\1/' /etc/nginx/sites-enabled/default
```

イメージをビルドしてコンテナを起動します。環境変数にInfluxDBのIPアドレスやデータベース名などの接続情報を指定します。

``` bash
$ docker build -t masato/grafana .
$ docker run --name grafana \
  -d \
  -p 8080:80 \
  -e INFLUXDB_HOST=10.1.3.67 \
  -e INFLUXDB_PORT=8086 \
  -e INFLUXDB_NAME=dweet  \
  -e INFLUXDB_USER=root \
  -e INFLUXDB_PASS=root \
  -e ELASTICSEARCH_HOST=10.1.3.67 \
  -e ELASTICSEARCH_PORT=9200 \
  -e ELASTICSEARCH_USER=admin \
  -e ELASTICSEARCH_PASS=mypass \
  masato/grafana
```

ログを確認して自動生成されたパスワードを確認します。

``` bash
$ docker logs grafana
=> Creating basic auth for "admin" user with random password
Adding password for user admin
=> Done!
========================================================================
You can now connect to Grafana with the following credential:

    admin:DUx0iSs3Zd10

========================================================================
=> Configuring InfluxDB
=> InfluxDB has been configured as follows:
   InfluxDB ADDRESS:  10.1.3.67
   InfluxDB PORT:     8086
   InfluxDB DB NAME:  sensortag
   InfluxDB USERNAME: root
   InfluxDB PASSWORD: root
   ** Please check your environment variables if you find something is misconfigured. **
=> Done!
=> Found Elasticsearch settings.
=> Set Elasticsearch url to "http://admin:mypass@10.1.3.67:9200".
=> Done!
=> Starting and running Nginx...
```

ブラウザで確認します。usernameと自動生成されたパスワードは以下です。

* username: admin
* password: DUx0iSs3Zd10

http://10.1.3.67:8080

![grafana.png](/2015/02/11/influxdb-grafana-dweet-example-on-docker/grafana.png)

## InfluxDBクライアントコンテナ

[dweet.io + InfluxDB + Grafana](http://datadventures.ghost.io/2014/09/07/dweet-io-influxdb-grafana/)を参考にして、カリフォルニアのアボガド果樹園の実際のデータを使ってテストをします。InfluxDBのPythonクライアントの[influxdb-python](https://github.com/influxdb/influxdb-python)を使いInfluxDBにレコードを登録していきます。Dockerのベースイメージは[google/python-runtime](https://registry.hub.docker.com/u/google/python-runtime/)を使います。

``` bash
$ mkdir -p ~/docker_apps/dwingest
$ cd !$
```

pipでインストールするパッケージを指定します。

```txt requirements.txt
influxdb
```

Dockerfileを作成します。ENTRYPOINTに実行するPythonスクリプトと引数を指定します。

``` bash Dockerfile
FROM google/python-runtime
ENTRYPOINT ["/env/bin/python", "/app/dwingest.py","10.1.3.67","AvocadoGrove","aiHotWaterTemp_degreesF","20"]
```

テスト用にInfluxDBのデータベースとユーザーをcurlを使い作成します。

* database: dweet
* username: dweet
* password: dweet

``` bash
$ curl -X POST 'http://10.1.3.67:8086/db?u=root&p=root' -d '{"name": "dweet"}'
$ curl -X POST 'http://10.1.3.67:8086/db/dweet/users?u=root&p=root' \
  -d '{"name": "dweet", "password": "dweet"}'
```

作業用コンテナを起動してdweet.ioからクエリしながらInfluxDBに登録していきます。

``` bash
$ docker run --rm --name dwingest dwingest
2015-02-10T07:12:27 Using InfluxDB host at 10.1.3.67:8086
2015-02-10T07:12:27 ================================================================================
2015-02-10T07:12:27 Querying thing AvocadoGrove with key aiHotWaterTemp_degreesF every 10 sec
2015-02-10T07:12:27 --------------------------------------------------------------------------------
2015-02-10T07:12:27 Starting new HTTP connection (1): 10.1.3.67
2015-02-10T07:12:28 Ingested value: 95.661
2015-02-10T07:12:38 --------------------------------------------------------------------------------
2015-02-10T07:12:38 Resetting dropped connection: 10.1.3.67
2015-02-10T07:12:38 Ingested value: 95.691
2015-02-10T07:12:48 --------------------------------------------------------------------------------
2015-02-10T07:12:49 Resetting dropped connection: 10.1.3.67
2015-02-10T07:12:49 Ingested value: 95.691
...
```

## Grafanaでグラフ作成

こんな感じで画面上で設定をしてグラフが簡単に作れます。

![grafana-edit.png](/2015/02/11/influxdb-grafana-dweet-example-on-docker/grafana-edit.png)
