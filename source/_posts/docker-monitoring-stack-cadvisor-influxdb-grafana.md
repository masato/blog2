title: "Docker監視環境 - Part3: cAdvisor, InfluxDB, Grafanaの構築"
date: 2014-10-08 03:36:29
tags:
 - Docker監視環境
 - MeasureAllTheThings
 - cAdvisor
 - InfluxDB
 - Grafana
 - StreamProcessing
 - Riemann
 - Heka
description: Riemannに続いて、cAdvisor, InfluxDB, Grafanaを使った監視環境を構築していきます。メトリクスを保存するバックエンドとして時系列データベースのInfluxDBの使い方を学習するのが目的です。Continuous Queriesを使って逐次的にデータを間引いたり、データの保持期間を決めたりできるようです。Measure All The Thingsを実践していて、Whisperのリテンションが間に合わなくなって自滅した経験があるので、RiemannやHekaを使ったメトリクスのStream Processingのバックエンドとして最適だと思います。
---

* Update: [DockerのInfluxDBとGrafanaでdweet.ioのデータを可視化する](/2015/02/11/influxdb-grafana-dweet-example-on-docker)

[Riemann](/2014/10/07/docker-monitoring-stack-riemann/)に続いて、cAdvisor, InfluxDB, Grafanaを使った監視環境を構築していきます。メトリクスを保存するバックエンドとして時系列データベースのInfluxDBの使い方を学習するのが目的です。[Continuous Queries](http://influxdb.com/docs/v0.8/api/continuous_queries.html)を使って逐次的にデータを間引いたり、データの保持期間を決めたりできるようです。`Measure All The Things`を実践していて、Whisperのリテンションが間に合わなくなって自滅した経験があるので、RiemannやHekaを使ったメトリクスの`Stream Processing`のバックエンドとして最適だと思います。


<!-- more -->

### InfluxDB

最初に[tutum-docker-influxdb](https://registry.hub.docker.com/u/tutum/influxdb)を使いInfluxDBを起動します。

``` bash
$ docker run --name influxdb \
  -d \
  -p 8083:8083 \
  -p 8086:8086 \
  --expose 8090 \
  --expose 8099 \
  -e PRE_CREATE_DB="cadvisor"  \
  tutum/influxdb
```

Dockerホスト(10.1.1.32)にポートマップします。

* コンソール   : 10.1.1.32:8083
* cAdvisor受信: 10.1.1.32:8086

ブラウザからInfluxDBのコンソールを表示してみます。

http://10.1.1.32:8083

* uesr: root
* passwd: root

{% img center /2014/10/08/docker-monitoring-stack-cadvisor-influxdb-grafana/influxdb.png %}

### cAdvidor

次にcAdvisorを起動します。`--storage_driver`を指定してバックエンドにInfluxDBを使用します。

Dockerホストにポートマップします。

* コンソール                : 10.1.1.32:8080
* cAdvidorからInfluxDBに接続: 10.1.1.32:8086

``` bash
$ docker run --name cadvisor \
  -d \
  -v /var/run:/var/run:rw \
  -v /sys:/sys:ro \
  -v /var/lib/docker/:/var/lib/docker:ro \
  -p 8080:8080 \
  google/cadvisor:latest \
  --storage_driver=influxdb \
  --storage_driver_host=10.1.1.32:8086 \
  --log_dir=/ 
```

ブラウザからInfluxDBのコンソールを開きます。

http://10.1.1.32:8080

{% img center /2014/10/08/docker-monitoring-stack-cadvisor-influxdb-grafana/cadvisor.png %}


### Grafana

[tutum/grafana](https://registry.hub.docker.com/u/tutum/grafana/)のイメージを利用しますが、この環境ではIPv6の場合エラーになりました。

``` bash
$ docker logs 040f3b9bfb04
=> Starting and running Nginx...
nginx: [emerg] socket() [::]:80 failed (97: Address family not supported by protocol)
```

IPv6をコメントアウトしたdefaultを作成して、新しいDockerイメージを用意します。

``` bash ~/docker_apps/grafana/default
server {
    listen 80 default_server;
    #listen [::]:80 default_server ipv6only=on;

    root /app;
    index index.html index.htm;

    server_name localhost;

    location /ping {
        return 200;
    }

    location / {
        auth_basic "Restricted";
        auth_basic_user_file /app/.htpasswd;
        try_files $uri $uri/ =404;
    }
}
```

Dockerfileを作成します。

``` bash ~/docker_apps/grafana/Dockerfile
FROM tutum/grafana
ADD default /etc/nginx/sites-enabled/default
```

Dockerイメージをbuildしてコンテナを起動します。

``` bash
$ docker build -t masato/grafana .
$ docker run --name grafana \
  -d \
  -p 8088:80 \
  -e INFLUXDB_HOST=10.1.1.32 \
  -e INFLUXDB_PORT=8086 \
  -e INFLUXDB_NAME=cadvisor \
  -e INFLUXDB_USER=root \
  -e INFLUXDB_PASS=root \
  masato/grafana
```

ログの確認すると、adminとパスワードが表示されます。

``` bash
$ docker logs grafana
=> Creating basic auth for " admin" user with random password
Adding password for user admin
=> Done!
========================================================================
You can now connect to Grafana with the following credential:

    admin:6eGfsVkrxSVh

========================================================================
=> Configuring InfluxDB
=> InfluxDB has been configured as follows:
   InfluxDB ADDRESS:  10.1.1.32
   InfluxDB PORT:     8086
   InfluxDB DB NAME:  cadvisor
   InfluxDB USERNAME: root
   InfluxDB PASSWORD: root
   ** Please check your environment variables if you find something is misconfigured. **
=> Done!
=> Either address or port of Elasticsearch is not set or empty.
=> Skip setting Elasticsearch.
=> Starting and running Nginx...
```

ブラウザでGrafanaのコンソールを開きます。

http://10.1.1.32:8088

ベーシック認証を入力します。

* user:   admin
* passwd: 6eGfsVkrxSVh
 
{% img center /2014/10/08/docker-monitoring-stack-cadvisor-influxdb-grafana/grafana.png %}

 
### まとめ

3つのコンテナはそれぞれ以下のアドレスで、Dockerホストにポートマップして起動しました。

cAdvisor : http://10.1.1.32:8080
InfluxDB : http://10.1.1.32:8083
Grafana  : http://10.1.1.32:8088


 