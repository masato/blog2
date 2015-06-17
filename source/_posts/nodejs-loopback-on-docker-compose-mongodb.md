title: "Node.js LoopBack on Docker - Part3: MongoDB connector"
date: 2015-06-09 14:17:42
tags:
 - LoopBack
 - StrongLoop
 - Nodejs
 - MongoDB
 - DockerCompose
description: 去年MBaaSのバックエンドの調査で少しLoopBackを使ってからしばらく触ってませんでした。今回は一般的なREST APIを公開するプロダクション環境用のフレームワークとして調査しています。alternativesにはExpress、restify、 Hapi.js、Koa、actionhero.jsといったフレームワークがあります。LoopBackの良い点はモデルを定義するだけでREST APIを公開できる手軽さにあります。他と比べて圧倒的に書くコード数が少ないです。Hapi.jsなどの軽量フレームワークより依存するライブラリが多くサイズも大きくなりますが、メリットは大きいです。
---

去年MBaaSの[バックエンド](/2014/12/25/nodejs-loopback-docker-getting-started-strongloop/)の調査で少し[LoopBack](https://strongloop.com/node-js/loopback-framework/)を使ってからしばらく触ってませんでした。今回は一般的なREST APIを公開するプロダクション環境用のフレームワークとして調査しています。alternativesには[Express](http://expressjs.com/)、[restify](https://github.com/restify/node-restify)、 [Hapi.js](http://hapijs.com/)、[Koa](https://github.com/koajs/koa)、[actionhero.js](http://www.actionherojs.com/)といったフレームワークがあります。LoopBackの良い点はモデルを定義するだけでREST APIを公開できる手軽さにあります。他と比べて圧倒的に書くコード数が少ないです。Hapi.jsなどの軽量フレームワークより依存するライブラリが多くサイズも大きくなりますが、メリットは大きいです。

<!-- more -->

## プロジェクト

LoopBackやMongoDBのコンテナをDocker Composeで構成管理します。コードは[リポジトリ](https://github.com/masato/docker-loopback)にpushしています。

```bash
$ cd ~/node_apps/docker-loopback
$ tree .
.
├── Dockerfile
├── README.md
└── docker-compose.yml

0 directories, 3 files
```

slcコマンドを非rootユーザーで実行してアプリ作成します。アプリのコードはDockerホストにマウントして編集したいので作業ユーザーを作ります。strongloopパッケージはグローバルインストールするので、nvmで仮想環境を用意してもよいですが今回はsudoをつけるようにしました。ユーザーはDockerホストの作業ユーザーと同じuidを指定します。

```bash ~/node_apps/docker-loopback/Dockerfile
FROM node:0.12
MAINTAINER Masato Shimizu <ma6ato@gmail.com>

RUN apt-get update && \
    apt-get install -y sudo && \
    rm -rf /var/lib/apt/lists/*

WORKDIR /app

RUN adduser --disabled-password --gecos '' --uid 1000 docker && \
  adduser docker sudo && \
  echo 'docker ALL=(ALL) NOPASSWD:ALL' >> /etc/sudoers && \
  chown -R docker:docker /app

RUN npm install -g --unsafe-perm strongloop
USER docker
ENTRYPOINT ["npm", "start"]
CMD []
```

Dockerイメージをビルドします。

```sh
$ docker build -t loopback .
```

slcコマンドからしてアプリを作成します。

```sh
$ docker-compose run --rm slc loopback

     _-----_
    |       |    .--------------------------.
    |--(o)--|    |  Let's create a LoopBack |
   `---------´   |       application!       |
    ( _´U`_ )    '--------------------------'
    /___A___\
     |  ~  |
   __'.___.'__
 ´   `  |° ´ Y `

? What's the name of your application? spike-todo
   create spike-todo/
     info change the working directory to spike-todo
...
Next steps:

  Change directory to your app
    $ cd spike-todo

  Create a model in your app
    $ slc loopback:model

  Optional: Enable StrongOps monitoring
    $ slc strongops

  Run the app
    $ slc run .

Removing dockerloopback_slc_run_1...
```

## アプリの設定

slcコマンドで作成したアプリ名をdocker-compose.ymlの`working_dir`に指定します。コンテナの作業ディレクトリは`/app`なので上記のアプリの場合は、`/app/spike-todo`になります。

MongoDBのコンテナはサービスを`mongo`とつけて`server`サービスからlinkします。`server`サービスの`/etc/hosts`に`mongo`の名前で自動的に登録されるためコンテナ間の名前解決が簡単になります。

```yaml ~/node_apps/docker-loopback/docker-compose.yml
slc: &defaults
  image: loopback
  volumes:
    - .:/app
  working_dir: /app/spike-todo
  links:
    - mongo
  entrypoint: ["slc"]
server:
  <<: *defaults
  entrypoint: ["slc","run"]
  ports:
    - "3000:3000"
npm:
  <<: *defaults
  entrypoint: ["npm"]
mongo:
  image: mongo
  volumes:
    - ./mongo:/data/db
```

### MondoDBサービス

docker-compose.ymlに定義したnpmサービスを使ってMongoDBコネクタの[loopback-connector-mongodb](https://github.com/strongloop/loopback-connector-mongodb)パッケージをインストールします。`--save`フラグをつけると`package.json`に依存関係を追記してくれます。

```sh
$ docker-compose run --rm npm install loopback-connector-mongodb --save
...
loopback-connector-mongodb@1.11.0 node_modules/loopback-connector-mongodb
├── debug@2.2.0 (ms@0.7.1)
├── loopback-connector@2.2.1
├── async@1.2.1
└── mongodb@2.0.33 (readable-stream@1.0.31, mongodb-core@1.1.32)
Removing dockerloopback_npm_run_1...
```

slcコマンドで作成したアプリの`spike-todo`ディレクトリにあるデータベース接続の設定ファイルを編集します。dbディレクティブはデフォルトで定義されています。mongodb_devという名前でディレクティブを作成します。


```json ~/node_apps/docker-loopback/spike-todo/server/datasources.json
  "db": {
    "name": "db",
    "connector": "memory"
  },
  "mongodb_dev": {
    "host": "mongo",
    "port": 27017,
    "database": "devDB",
    "username": "devUser",
    "password": "",
    "name": "",
    "connector": "mongodb"
  }
}
```

### モデルの作成

MongoDBコネクタをインストールして、datasources.jsonにエントリを追加したのでslcコマンドのdata-sourceの選択にmongodbが表示されます。ベースクラスはPersistedModelを選択してデータベースのCRUD操作ができるようにします。続くモデルのプロパティ設定はToDoアプリっぽくstringの`titie`とbooleanの`created`の2つを追加します。

```sh
$ docker-compose run --rm slc loopback:model
? Enter the model name: Todo
? Select the data-source to attach Todo to: mongodb_dev (mongodb)
? Select model's base class: PersistedModel
? Expose Todo via the REST API? Yes
? Custom plural form (used to build REST URL): Todos
Let's add some Todo properties now.

Enter an empty property name when done.
? Property name: title
   invoke   loopback:property
? Property type: string
? Required? Yes

Let's add another Todo property.
Enter an empty property name when done.
? Property name: completed
   invoke   loopback:property
? Property type: boolean
? Required? No

Let's add another Todo property.
Enter an empty property name when done.
? Property name:
Removing dockerloopback_slc_run_1...
```

### サーバーの起動

`docker-compose up`からサービスを起動するとdocker-compose.ymlに定義したホストへのポートマップが有効になります。Dockerホストの外側からはDockerホストのIPアドレスを指定して`http://xxx..xxx.xxx:3000/explorer`をブラウザで開きます。

```sh
$ docker-compose up server
Creating dockerloopback_mongo_1...
Creating dockerloopback_server_1...
Attaching to dockerloopback_server_1
server_1 | INFO strong-agent v1.6.0 profiling app 'spike-todo' pid '1'
server_1 | INFO strong-agent[1] started profiling agent
server_1 | INFO supervisor reporting metrics to `internal:`
server_1 | supervisor running without clustering (unsupervised)
server_1 | INFO strong-agent not profiling, agent metrics requires a valid license.
server_1 | Please contact sales@strongloop.com for assistance.
server_1 | Browse your REST API at http://0.0.0.0:3000/explorer
server_1 | Web server listening at: http://0.0.0.0:3000/
```

## Swagger UIとREST APIの実行

3000ポートを開くとLoopBackサーバーのuptimeが表示されます。

```bash
$ curl http://localhost:3000
{"started":"2015-06-09T02:31:05.307Z","uptime":37.229}
```

DBの設定ファイルを定義してslcコマンドからモデルを作成しただけで、ここまでコードは1行も書いていませんがこれだけで[Swagger UI](https://github.com/swagger-api/swagger-ui)とREST APIが使えるようになります。初めてRailsを触ったときのようなちょっとした感動があります。Swagger UIの画面上からAPIの実行をすることもできます。

![todo-create.png](/2015/06/09/nodejs-loopback-on-docker-compose-mongodb/todo-create.png)

今回はcurlでREST APIを実行します。JSON形式でPOSTしてレコードを作成します。

```js
$ curl -X POST -H "Content-Type:application/json" \
-d '{"title": "サンプル", "completed": false}' \
http://localhost:3000/api/ToDos
{"title":"サンプル","completed":false,"id":"55766dcba3148801001e6e42"}
```

今回はクエリ文字列に日本語を使っているので、素ではURIエンコードは書けませんがGETメソッドを使いフィルタしたレコードを取得することができます。

```sh
$ curl -X GET localhost:3000/api/Todos?filter=%7B%22where%22%3A%7B%22title%22%3A%22%E3%82%B5%E3%83%B3%E3%83%97%E3%83%AB%22%7D%7D
[{"title":"サンプル","completed":false,"id":"55766dcba3148801001e6e42"}]
```

上記のクエリ文字列は以下のWhere句をURIエンコードしたものです。Swagger UIの場合クエリをフィールドに入力すれば自動的にURIエンコードしたURLを生成して実行してくれます。

```js
{"where":{"title":"サンプル"}}
```
![filter.png](/2015/06/09/nodejs-loopback-on-docker-compose-mongodb/filter.png)

念のため起動中のMongoDBコンテナに入りレコードが追加されたことを確認します。

```sh
$ docker exec -it dockerloopback_mongo_1 mongo devDB
MongoDB shell version: 3.0.3
connecting to: devDB
> db.Todo.find({title:"サンプル"});
{ "_id" : ObjectId("55766dcba3148801001e6e42"), "title" : "サンプル", "completed" : false }
```
