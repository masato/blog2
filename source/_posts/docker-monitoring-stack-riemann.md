title: "Docker監視環境 - Part2: Riemannの構築"
date: 2014-10-07 01:35:44
tags:
 - Docker監視環境
 - MeasureAllTheThings
 - Riemann
 - Clojure
 - StreamProcessing
 - Ruby
 - clostack
description: Docker監視環境 - Part1 cAdvisor, collectd, Riemann, InfluxDB, Grafanaで決めたアーキテクチャのPoCをしていきます。最初にRiemannを構築します。RiemannはClojureらしい分散システムの監視ツールです。似たようなストリーム処理のフレームワークにLaminaやStormがあります。Stormのようなクラスタでビッグデータを処理するほどでもなく、Do-it-yourself CloudWatch-style alarms using Riemannにあるような用途が主です。system metricsとapplication metricsの収集 ブラウザで可視化とクエリ実行 閾値を監視してnotificationやtriggerを実行 clostackを使ってCloudStackでオートスケールなんておもしろうそうです。
---

* `Update 2014-10-08`: [Docker監視環境 - Part3: cAdvisor,InfluxDB,Grafanaの構築](/2014/10/08/docker-monitoring-stack-cadvisor-influxdb-grafana/)

[Docker監視環境 - Part1: cAdvisor, collectd, Riemann, InfluxDB, Grafana](/2014/09/26/docker-monitoring-stack-prepare/)で決めたアーキテクチャのPoCをしていきます。最初に[Riemann](https://github.com/aphyr/riemann)を構築します。
RiemannはClojureらしい分散システムの監視ツールです。似たようなストリーム処理のフレームワークに[Lamina](https://github.com/ztellman/lamina)や[Storm](https://github.com/nathanmarz/storm)があります。
Stormのようなクラスタでビッグデータを処理するほどでもなく、[Do-it-yourself CloudWatch-style alarms using Riemann](http://cloudierthanthou.wordpress.com/2014/01/09/do-it-yourself-cloudwatch-style-alarms-using-riemann/)にあるような用途が主です。

* `system metrics`と`application metrics`の収集
* ブラウザで可視化とクエリ実行
* 閾値を監視してnotificationやtriggerを実行

[clostack](https://github.com/pyr/clostack)を使ってCloudStackでオートスケールなんておもしろうそうです。

<!-- more -->

### プロジェクトの作成

RiemannのDockerイメージを作成するプロジェクトの構成です。

``` bash
$ tree ~/docker_apps/riemann
~docker_apps/riemann
├── Dockerfile
└── riemann.config
```

ダッシュボードの[Riemann-Dash](https://github.com/aphyr/riemann-dash)イメージを作成するプロジェクトです。

``` bash
$ tree ~/docker_apps/riemann-health/
~/docker_apps/riemann-health/
├── Dockerfile
├── Gemfile
└── Gemfile.lock
```

もう一つRubyで書かれた`system metrics`を収集するriemann-healthを使うため、[Riemann Tools](https://github.com/aphyr/riemann-tools)のコンテナを用意します。


### RiemannのDockerfile

[patrickod/riemann](httpのs://github.com/patrickod/riemann-docker)を参考にしてDockerfileを書いていきます。
`2014-10-06`現在で最新バージョンは`0.2.6`です。

``` bash ~/docker_apps/riemann/Dockerfile
ROM ubuntu:14.04
MAINTAINER Masato Shimizu <ma6ato@gmail.com>
## apt-get update
RUN sed -i~ -e 's/archive.ubuntu.com/ftp.jaist.ac.jp/' /etc/apt/sources.list \
 && apt-get update
RUN DEBIAN_FRONTEND=noninteractive apt-get install -y curl default-jre-headless
# Download the latest .deb and install
RUN curl http://aphyr.com/riemann/riemann_0.2.6_all.deb > /tmp/riemann_0.2.6_all.deb
RUN dpkg -i /tmp/riemann_0.2.6_all.deb
# Expose the ports for inbound events and websockets
EXPOSE 5555
EXPOSE 5555/udp
EXPOSE 5556
# Share the config directory as a volume
VOLUME /etc/riemann
ADD riemann.config /etc/riemann/riemann.config
# Set the hostname in /etc/hosts so that Riemann doesn't die due to unknownHostException
RUN echo 127.0.0.1 $(hostname) > /etc/hosts
CMD ["/usr/bin/riemann","/etc/riemann/riemann.config"]
```

### riemann.config

設定ファイルは試行錯誤でなるべく単純に書きました。Dockerで動かすのでサーバーは`0.0.0.0`にバインドして、Dockerの外からアクセスできるようにします。

``` clj ~/docker_apps/riemann/riemann.config
(logging/init {:file "/etc/riemann/riemann.log" :console true})

(let [host "0.0.0.0"]
  (tcp-server :host host)
  (udp-server :host host)
  (ws-server :host host))

(periodically-expire 5)

(let [index (tap :index (index))]
  (streams
    index
    (with {:metric 1 :host nil :state "ok" :service "events/sec"}
      (rate 5 index))
    (expired
      (fn [event] (info "expired" event)))))

(streams
        prn)
```

### Riemannコンテナの起動

イメージをビルドしてRiemannコンテナを起動します。

``` bash
$ cd ~/docker_apps/riemann
$ docker build -t masato/riemann .
$ docker run -it --name riemann --rm -v $(pwd):/etc/riemann -p 5556:5556 -p 5555:5555  masato/riemann
```

### Riemann-Dashコンテナの起動

[Riemann-Dash](https://github.com/aphyr/riemann-dash)はSinatraのアプリです。Thinサーバーで起動しています。

``` bash
$ docker run --name riemann-dash --rm -p 4567:4567 davidkelley/riemann-dash
== Sinatra/1.4.5 has taken the stage on 4567 for development with backup from Thin
```

### Riemann Toolsコンテナの起動

[Docker Adds 11 Top Language Stacks to Docker Hub Registry](http://www.businesswire.com/news/home/20140924005223/en/Docker-Adds-11-Top-Language-Stacks-Docker)によるとRubyのオフィシャルイメージも追加されたので、[Riemann Tools](https://github.com/aphyr/riemann-tools)用のDockerfileで使ってみます。

`riemann-health run`は`--host`オプションでDockerホストのIPアドレスを指定します。

``` bash ~/docker_apps/riemann-health/Dockerfile
FROM ruby:2.1.3-onbuild
RUN gem install riemann-tools --no-doc
CMD ["riemann-health","--host","10.1.1.32"]
```

ONBUILDで追加されるGemfileで`riemann-tools`をインストールします。

``` bash ~/docker_apps/riemann-health/Gemfile
source 'https://rubygems.org'
gem 'riemann-tools'
```

同じくONBUILDで追加されるGemfile.lockです。

``` bash ~/docker_apps/riemann-health/Gemfile.lock
GEM
  remote: https://rubygems.org/
  specs:
    beefcake (1.0.0)
    builder (3.2.2)
    excon (0.40.0)
    faraday (0.9.0)
      multipart-post (>= 1.2, < 3)
    fog (1.23.0)
      fog-brightbox
      fog-core (~> 1.23)
      fog-json
      fog-softlayer
      ipaddress (~> 0.5)
      nokogiri (~> 1.5, >= 1.5.11)
    fog-brightbox (0.5.1)
      fog-core (~> 1.22)
      fog-json
      inflecto
    fog-core (1.24.0)
      builder
      excon (~> 0.38)
      formatador (~> 0.2)
      mime-types
      net-scp (~> 1.1)
      net-ssh (>= 2.1.3)
    fog-json (1.0.0)
      multi_json (~> 1.0)
    fog-softlayer (0.3.19)
      fog-core
      fog-json
    formatador (0.2.5)
    inflecto (0.0.2)
    ipaddress (0.8.0)
    mime-types (2.3)
    mini_portile (0.6.0)
    mtrc (0.0.4)
    multi_json (1.10.1)
    multipart-post (2.0.0)
    munin-ruby (0.2.5)
    net-scp (1.2.1)
      net-ssh (>= 2.6.5)
    net-ssh (2.9.1)
    nokogiri (1.6.3.1)
      mini_portile (= 0.6.0)
    riemann-client (0.2.3)
      beefcake (>= 0.3.5)
      mtrc (>= 0.0.4)
      trollop (>= 1.16.2)
    riemann-tools (0.2.2)
      faraday (>= 0.8.5)
      fog (>= 1.4.0)
      munin-ruby (>= 0.2.1)
      nokogiri (>= 1.5.6)
      riemann-client (>= 0.2.2)
      trollop (>= 1.16.2)
      yajl-ruby (>= 1.1.0)
    trollop (2.0)
    yajl-ruby (1.2.1)

PLATFORMS
  ruby

DEPENDENCIES
  riemann-tools
```

buildとrunを実行します。

``` bash
$ docker build -t masato/riemann-health .
...
Step 3 : CMD ["riemann-health","--host","10.1.1.32"]
 ---> Running in f5a6e5cfcacb
 ---> bfa01e66fe51
Removing intermediate container f5a6e5cfcacb
Successfully built bfa01e66fe51
$ docker run --name riemann-health --rm -it masato/riemann-health
```

### IPアドレスの確認

コンテナのIPアドレスの確認をします。

``` bash
$ docker inspect --format="&#123;&#123; .NetworkSettings.IPAddress }}" riemann
172.17.1.12
$ docker inspect --format="&#123;&#123; .NetworkSettings.IPAddress }}" riemann-dash
172.17.1.13
$ docker inspect --format="&#123;&#123;.NetworkSettings.IPAddress }}" riemann-health
172.17.1.24
```

### Riemann-Dash確認

ブラウザでThinで起動しているSinatraを表示します。

http://172.17.1.13:4567

以下の手順ですべてのメトリクスを表示する設定をいれます。

1. 大きなRiemannの文字をクリックしてアクティブにする
2. キーボードのeを押す
3. セレクトボックスからGridを選択
4. queryフィールドにtrueを入れる
5. Applyボタンを押す

{% img center /2014/10/07/docker-monitoring-stack-riemann/riemann.png %}


4ebef3433a61が`riemann-health`で`system metrics`をriemannに送信しているコンテナです。
ae55fc1b437dgがriemannサーバーが起動しているコンテナです。

最後の行のnilはよくわかりません。riemann.configの設定が間違っている気がしますが。

``` bash
$ docker ps
CONTAINER ID        IMAGE                             COMMAND                CREATED             STATUS              PORTS                                                      NAMES
4ebef3433a61        masato/riemann-health:latest      "riemann-health --ho   37 minutes ago      Up 37 minutes                                                                  riemann-health
bd3ff7613a4f        davidkelley/riemann-dash:latest   "riemann-dash"         About an hour ago   Up About an hour    0.0.0.0:4567->4567/tcp                                     riemann-dash
ae55fc1b437d        0ff641514409                      "/usr/bin/riemann /e   About an hour ago   Up About an hour    5555/udp, 0.0.0.0:5555->5555/tcp, 0.0.0.0:5556->5556/tcp   riemann
```










