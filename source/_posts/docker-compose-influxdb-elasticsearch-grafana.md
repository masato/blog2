title: "InfluxDBとElasticSearchとGrafanaのためdocker-compose.yml"
date: 2015-06-03 20:07:12
tags:
 - InfluxDB
 - ElasticSerach
 - Grafana
 - DockerCompose
 - DHT11
description: 温湿度センサのDHT11で計測した温度と湿度をダッシュボードに表示するための環境を用意します。以前dweet.ioからアボガド果樹園のデータを取得したときにDockerコンテナで構築しました。このときはコンテナを個別に作成したのでコンテナの起動が煩雑でした。Docker Composeを使うと1つのYAMLに設定をまとめてdocker-compose upするだけで簡単です。また定義したサービス名がそれぞれの/etc/hostsに設定してくれます。環境変数に設定するホスト名はこのサービス名を使うことができるのでIPアドレスをinspectしてハードコードする必要がありません。
---

温湿度センサのDHT11で計測した温度と湿度をダッシュボードに表示するための環境を用意します。以前dweet.ioからアボガド果樹園のデータを取得したときに[Dockerコンテナで構築](/2015/02/11/influxdb-grafana-dweet-example-on-docker/)しました。このときはコンテナを個別に作成したのでコンテナの起動が煩雑でした。Docker Composeを使うと1つのYAMLに設定をまとめて`docker-compose up`するだけで簡単です。また定義したサービス名がそれぞれの`/etc/hosts`に設定してくれます。環境変数に設定するホスト名はこのサービス名を使うことができるのでIPアドレスをinspectしてハードコードする必要がありません。

<!-- more -->

## プロジェクトの作成

適当なディレクトリを作成してdocker-compose.ymlを書くだけです。このディレクトリにはInfluxDBのボリュームをアタッチされます。

```bash
$ cd ~/influx_apps
$ tree
.
└── docker-compose.yml
```

## docker-compose.yml

以下のdocker-compose.ymlを作成してinfluxdb、elasticsearch、grafanaのサービスを定義します。すべて[Tutum](https://www.tutum.co/)がDockerイメージを提供してくれます。

* [tutum/influxdb](https://registry.hub.docker.com/u/tutum/influxdb/)
* [tutum/elasticsearch](https://registry.hub.docker.com/u/tutum/elasticsearch/)
* [tutum/grafana](https://registry.hub.docker.com/u/tutum/grafana/)

Grafanaのポートだけ他のサービスと競合するのでDockerホストには80から変更して8080にマップしています。

```yaml ~/influx_apps/docker-compose.yml
influxdb:
  restart: always
  image: tutum/influxdb
  volumes:
    - ./influxdb:/data
  environment:
    - PRE_CREATE_DB=influxdb
  ports:
    - "8083:8083"
    - "8086:8086"
elastic:
  restart: always
  image: tutum/elasticsearch
  volumes:
    - ./elastic:/usr/share/elasticsearch/data
  environment:
    - ELASTICSEARCH_USER=admin
    - ELASTICSEARCH_PASS=mypass
  ports:
    - "9200:9200"
grafana:
  restart: always
  image: tutum/grafana
  ports:
    - "8080:80"
  environment:
    - INFLUXDB_PROTO=http
    - INFLUXDB_HOST=influxdb
    - INFLUXDB_PORT=8086
    - INFLUXDB_NAME=dht11
    - INFLUXDB_USER=root
    - INFLUXDB_PASS=root
    - ELASTICSEARCH_PROTO=http
    - ELASTICSEARCH_HOST=elastic
    - ELASTICSEARCH_PORT=9200
    - ELASTICSEARCH_USER=admin
    - ELASTICSEARCH_PASS=mypass
```

## フォアグラウンドで起動

コンテナをまとめて起動します。起動シーケンスを確認したいのでフォアグラウンドでテスト起動します。

``` bash
$ cd ~/influx_apps
$ docker-compose up
```

### InfluxDBのテスト

コンテナのbashを実行してInfluxDBのバージョンを確認します。`0.8.8`を使っています。

``` bash
$ docker exec -it influxapps_influxdb_1 bash
$ influxdb -v
InfluxDB v0.8.8 (git: afde71e) (leveldb: 1.15)
```

[Reading and Writing Data](http://influxdb.com/docs/v0.8/api/reading_and_writing_data.html)に書いてあるようにテストデータを投入してみます。Dockerホストからcurlで確認します。

``` bash
$ curl -X POST -d '[{"name":"foo","columns":["val"],"points":[[23]]}]' 'http://localhost:8086/db/influxdb/series?u=root&p=root'
```

レコードをクエリしてみます。POSTしていないデータのtimeとsequence_numberは自動的に入ります。

```bash
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
                1433443454781,
                10001,
                23
            ]
        ]
    }
]
```

リモートからブラウザでDockerホストにアクセスしてInfuxDBの管理画面を表示します。

http://xxx.xxx.xxx.xxx:8083/

![influxdb.png](/2015/06/03/docker-compose-influxdb-elasticsearch-grafana/influxdb.png)

### ElasticSearchのテスト

DockerホストからElasticSerchの起動確認を兼ねてバージョンを表示します。`1.3.9`を使っています。

```bash
$ curl admin:mypass@localhost:9200
{
  "status" : 200,
  "name" : "Abner Jenkins",
  "version" : {
    "number" : "1.3.9",
    "build_hash" : "0915c7306e6264ba21a6cb7609b93545ccc32ef1",
    "build_timestamp" : "2015-02-19T12:34:48Z",
    "build_snapshot" : false,
    "lucene_version" : "4.9"
  },
  "tagline" : "You 
```

### Grafanaのテスト
 
Grafanaは2015/04/20に2.0がリリースされていますが、今回のイメージは`1.9.1`を使っています。

``` bash
$ docker exec -it influxapps_grafana_1 bash
$ echo $GRAFANA_VERSION
1.9.1
```

Grafanaの管理画面にアクセスするためランダムで生成されるパスワードを標準出力のログから確認します。

* login: admin
* password: 5jG0HLZ09sZQ

``` bash
...
grafana_1 | => Creating basic auth for "admin" user with random password
grafana_1 | Adding password for user admin
grafana_1 | => Done!
grafana_1 | ========================================================================
grafana_1 | You can now connect to Grafana with the following credential:
grafana_1 |
grafana_1 |     admin:5jG0HLZ09sZQ
grafana_1 |
grafana_1 | ========================================================================
grafana_1 | => Configuring InfluxDB
grafana_1 | => InfluxDB has been configured as follows:
grafana_1 |    InfluxDB ADDRESS:  influxdb
grafana_1 |    InfluxDB PORT:     8086
grafana_1 |    InfluxDB DB NAME:  dht11
grafana_1 |    InfluxDB USERNAME: root
grafana_1 |    InfluxDB PASSWORD: root
grafana_1 |    ** Please check your environment variables if you find something is misconfigured. **
grafana_1 | => Done!
grafana_1 | => Found Elasticsearch settings.
grafana_1 | => Set Elasticsearch url to "http://admin:mypass@elastic:9200".
grafana_1 | => Done!
grafana_1 | => Starting and running Nginx...
```

リモートからGrafanaのダッシュボードをブラウザで開きます。通常80ポートですが他のサービスと競合するのでDockerホストには8080でマップしています。

http://xxx.xxx.xxx.xxx:8080/

![grafana.png](/2015/06/03/docker-compose-influxdb-elasticsearch-grafana/grafana.png)


## バックグラウンドで起動

正常に動作しているようなので、コンテナを停止してボリュームも削除してからバックグラウンドで起動します。

``` bash
$ docker-compose stop
Stopping influxapps_grafana_1...
Stopping influxapps_elastic_1...
Stopping influxapps_influxdb_1...
```

コンテナを削除します。

``` bash
$ docker-compose rm
Going to remove influxapps_grafana_1, influxapps_elastic_1, influxapps_influxdb_1
Are you sure? [yN] y
Removing influxapps_influxdb_1...
Removing influxapps_elastic_1...
$ sudo rm -fr influxdb/ elastic/
```

バックグラウンドで起動します。

```bash
$ docker-compose up -d
Creating influxapps_influxdb_1...
Creating influxapps_elastic_1...
Creating influxapps_grafana_1...
```

ブラウザで管理画面のBASIC認証に必要なGrafanaのパスワードは`docker-compose logs`コマンドで確認します。

```bash
$ docker-compose logs grafana
```


