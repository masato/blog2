title: "MeshbluでオープンソースのIoTをはじめよう - Part4: Hubot with Slack on Docker"
date: 2015-07-13 19:12:36
categories:
 - IoT
tags:
 - Meshblu
 - Hubot
 - Slack
 - DockerCompose
 - MQTT
description: これまでにRaspberry Piと環境センサーから計測したデータをMeshbluのMQTTブローカーにpublishしたあとはfreeboardのダッシュボードに表示しました。今回はHubotとSlackを使ってインタラクティブなインタフェースを追加しようと思います。SlackはモダンなUIでWebhookや他のWebサービスとのインテグレーションがしやすいので便利に使えます。
---

[これまで](/2015/07/07/meshblu-compose-bme280-freeboard/)にRaspberry Piと環境センサーから計測したデータを[Meshblu](https://github.com/octoblu/meshblu/)のMQTTブローカーにpublishしたあとは[freeboard](https://github.com/Freeboard/freeboard)のダッシュボードに表示しました。今回は[Hubot](https://github.com/github/hubot)と[Slack](https://slack.com/)を使ってインタラクティブなインタフェースを追加しようと思います。SlackはモダンなUIでWebhookや他のWebサービスとのインテグレーションがしやすいので便利に使えます。

<!-- more -->

## HubotとSlackの設定

SlackのIntegrationページ`https://{teamdomain}.slack.com/services/new`からHubotを選び`API Token`を作成しておきます。現在ではSlackとHubotのインテグレーションに必要な情報はこの`API Token`だけになりとても簡単です。


![hubot-avodado.png](/2015/07/13/meshblu-compose-hubot-slack/hubot-avodado.png)


botには名前を付けてアイコンを変更することができます。今回は`avodado`と名付けました。

## プロジェクト

今回作成したプロジェクトのディレクトリ構成です。リポジトリは[こちら](https://github.com/IDCFChannel/docker-hubot-slack)です。

```bash
$ cd ~/node_apps
$ git clone https://github.com/IDCFChannel/docker-hubot-slack
$ cd docker-hubot-slack
$ tree
.
├── Dockerfile
├── README.md
├── docker-compose.yml
├── docker-compose.yml.default
├── redis
└── scripts
    └── hello.coffee
```

### Dockerfile

ベースイメージはオフィシャルの[io.js](https://registry.hub.docker.com/_/iojs/)を使います。Hubotの動作に必要な基本的なnpmはグルーバルにrootでインストールします。Dockerイメージの中にbotプロジェクトをYeomanで作成しておきます。コンテナのscriptsディレクトリはDockerホストのディレクトリをマウントして使います。

今回は単純にscriptsディレクトリ配下に直接CoffeeScriptのファイルを配置するだけで、external-scripts.jsonは使いません。

```bash ~/node_apps/docker-hubot-slack/scripts/Dockerfile
FROM iojs:2.3
MAINTAINER Masato Shimizu <ma6ato@gmail.com>

RUN mkdir -p /app
WORKDIR /app

RUN adduser --disabled-password --gecos '' --uid 1000 docker && \
    chown -R docker:docker /app

RUN npm install -g hubot coffee-script yo generator-hubot

USER docker
RUN yes | yo hubot --defaults && \
    npm install --save hubot-slack mqtt
```

### docker-compose.yml

docker-compose.yml.defaultをリネームして使います。

```bash
$ mv docker-compose.yml.default docker-compose.yml
```

Dockerイメージの`/app`ディレクトリにはYeomanを使ってbotを作ってあります。scriptsディレクトリをDockerホストから編集できるようにマウントします。環境変数にはテスト用にHUBOT_LOG_LEVELをデバッグレベルにして、HUBOT_SLACK_TOKENにはSlackのIntegrationページで取得したHubotのAPI Tokenを記入します。

また、Hubotの動作にはRedisが必要になるのでredisサービスを追加してnpmサービスにlinkします。


```yml ~/node_apps/docker-hubot-slack/scripts/docker-compose.yml
npm:
  build: .
  volumes:
    - ./scripts:/app/scripts
    - /etc/localtime:/etc/localtime:ro
  ports:
    - 8089:8089
  environment:
    - PORT=8089
    - REDIS_URL=redis://redis:6379
    - HUBOT_LOG_LEVEL=debug
    - HUBOT_SLACK_TOKEN=xxx
  links:
    - redis
  command: ./bin/hubot -a slack
redis:
  image: redis
  volumes:
    - ./redis:/data
    - /etc/localtime:/etc/localtime:ro
```

## 使い方



### ビルドと実行

Dockerイメージをビルドします。npmサービスをビルドしてupします。

```bash
$ cd ~/node_apps/docker-hubot-slack
$ docker-compose build npm
$ docker-compose up npm
```

### scripts/hello.coffee

helloとbotに発言すると、`Hi`と返答してくる単純なサンプルです。今回はこのhelloサンプルのbotスクリプトを実行してみます。

```coffeescript ~/node_apps/docker-hubot-slack/scripts/hello.coffee
module.exports = (robot) ->
  robot.respond /HELLO$/i, (res) ->
    res.reply "Hi"
```

Slackにログインして`avocado hello`と発言すると、`masato: Hi`と返答してくれます。

![hello-hi.png](/2015/07/13/meshblu-compose-hubot-slack/hello-hi.png)
