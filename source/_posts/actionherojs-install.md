title: "actionhero.js入門 - Part1: Getting Started"
date: 2015-06-20 02:40:51
tags:
 - actionherojs
 - Nodejs
 - WebSocket
 - Ajax
 - API
description: actionhero.jsはNode.jsのAPIフレームワークです。TCPソケット、WebSocket、Ajaxのマルチトランスポートに対応しているのが特徴です。チュートリアルやライブコーディング動画なども充実しているので勉強の助けになりそうです。
---

[actionhero.js](http://www.actionherojs.com)はNode.jsのAPIフレームワークです。TCPソケット、WebSocket、Ajaxのマルチトランスポートに対応しているのが[特徴](http://www.actionherojs.com/docs/#who-is-the-actionhero)です。チュートリアルやライブコーディング動画なども充実しているので勉強の助けになりそうです。

<!-- more -->

## 特徴

### マルチトランスポート

actionheroの名前の通り、1つのエンドポイントで複数のトランスポートプロトコルに対応したAPIのアクションが作れます。最近のクライアントアプリは特にSPAになるとREST API/AjaxとWebSocket/Socket.IOをどちらも使うことが多いです。


### タスクの非同期処理

APIサーバーとしての機能の他にRedisを使ったクラスター機能や、豊富なミドルウェアが提供されています。特に使ってみたいミドルウェアは[node-resque](https://github.com/taskrabbit/node-resque)を使ったタスクの非同期処理です。

[Sails.js](http://sailsjs.org/)もミドルウェアで機能を拡張できるようですが、actionhero.jsの場合はデフォルトで必要な機能が組み込まれているので便利です。

### Alternatives

APIフレームワークとしては以下がalternativesになります。

* [Express](http://expressjs.com/)
* [restify](https://github.com/restify/node-restify)
* [Hapi.js](http://hapijs.com/)
* [Koa](https://github.com/koajs/koa)
* [LoopBack](https://github.com/strongloop/loopback)
* [Sails.js](http://sailsjs.org/)


## Getting Started

さっそく[Getting Started](http://www.actionherojs.com/docs/ops/getting-started.html)を実行する環境を用意します。

### プロジェクト

適当なディレクトリにプロジェクト作成します。

```
$ cd ~/node_apps/docker_actionhero
$ tree
.
├── Dockerfile
├── docker-compose.yml
└── node_modules -> /dist/node_modules
```

コンテナ内で`/app/node_modules`が`/dist/node_modules`のシムリンクになっているので、カレントディレクトリをコンテナの`/app`ディレクトリにマップしたときも同様にシムリンクが動作するようにしています。

```bash
$ ln -s /dist/node_modules .
```

Dockerfileには作業ユーザーを作成します。今回はpackage.jsonを使わずにnpmコマンドでactionheroをグローバルにインストールします。後ほどactionheroコマンドでコードをジェネレーとしますが、カレントディレクトリにpackage.jsonを生成するため空けておきます。


```bash:~/node_apps/docker-actionhero/Dockerfile
FROM node:0.12
MAINTAINER Masato Shimizu <ma6ato@gmail.com>

WORKDIR /app

RUN adduser --disabled-password --gecos '' --uid 1000 docker && \
    mkdir -p /dist/node_modules && \
    ln -s /dist/node_modules /app/node_modules && \
    chown -R docker:docker /app /dist/node_modules

USER docker
RUN npm install actionhero

COPY . /app
ENTRYPOINT ["npm", "start"]
CMD []
```

Docker ComposeでRedisも一緒にインストールします。linksディレクティブで連係できるので簡単です。actionhero.jsはデフォルトは8080でserveします。Dockerホストの8080ポートはすでに使ってしまっているので、ここでは8089にマップします。

```yml:~/node_apps/docker-actionhero/docker-compose.yml
server:  &defaults
  image: masato/actionhero
  volumes:
    - .:/app
    - /etc/localtime:/etc/localtime:ro
  ports:
    - 8089:8080
  links:
    - redis
actionhero:
  <<: *defaults
  entrypoint: ["./node_modules/.bin/actionhero"]
npm:
  <<: *defaults
  entrypoint: ["npm"]
bash:
  <<: *defaults
  entrypoint: ["bash"]
redis:
  image: redis
  restart: always
  volumes:
    - ./redis:/data
```

Dockerイメージをビルドします。

```bash
$ docker build -t masato/actionhero .
```

Dockerfile内のnpm installでactionheroコマンドが`./node_modules/.bin/actionhero`にインストールされました。generateコマンドからアプリのテンプレートを作成します。


```bash
$ docker-compose run --rm actionhero generate
info: actionhero >> generate
info: Generating a new actionhero project...
info:  - creating directory '/app/actions'
info:  - creating directory '/app/pids'
info:  - creating directory '/app/config'
info:  - creating directory '/app/config/servers'
info:  - creating directory '/app/config/plugins'
info:  - creating directory '/app/initializers'
info:  - creating directory '/app/log'
info:  - creating directory '/app/servers'
info:  - creating directory '/app/public'
info:  - creating directory '/app/public/javascript'
info:  - creating directory '/app/public/css'
info:  - creating directory '/app/public/logo'
info:  - creating directory '/app/tasks'
info:  - creating directory '/app/test'
info:  - wrote file '/app/config/api.js'
info:  - wrote file '/app/config/plugins.js'
info:  - wrote file '/app/config/logger.js'
info:  - wrote file '/app/config/redis.js'
info:  - wrote file '/app/config/stats.js'
info:  - wrote file '/app/config/tasks.js'
info:  - wrote file '/app/config/errors.js'
info:  - wrote file '/app/config/routes.js'
info:  - wrote file '/app/config/servers/socket.js'
info:  - wrote file '/app/config/servers/web.js'
info:  - wrote file '/app/config/servers/websocket.js'
info:  - wrote file '/app/package.json'
info:  - wrote file '/app/actions/status.js'
info:  - wrote file '/app/actions/showDocumentation.js'
info:  - wrote file '/app/public/index.html'
info:  - wrote file '/app/public/chat.html'
info:  - wrote file '/app/public/css/actionhero.css'
info:  - wrote file '/app/public/logo/actionhero.png'
info:  - wrote file '/app/public/logo/sky.jpg'
info:  - file '/app/README.md' already exists, skipping
info:  - wrote file '/app/gruntfile.js'
info:  - wrote file '/app/test/example.js'
info:
info: Generation Complete.  Your project directory should look like this:
|- config
| -- api.js
| -- logger.js
| -- redis.js
| -- stats.js
| -- tasks.js
| -- servers
| ---- web.js
| ---- websocket.js
| ---- socket.js
|-- (project settings)
|
|- actions
|-- (your actions)
|
|- initializers
|-- (any additional initializers you want)
|
|- log
|-- (default location for logs)
|
|- node_modules
|-- (your modules, actionhero should be npm installed in here)
|
|- pids
|-- (pidfiles for your running servers)
|
|- public
|-- (your static assets to be served by /file)
|
|- servers
|-- (custom servers you may make)
|
|- tasks
|-- (your tasks)
|
|- tests
|-- (tests for your API)
|
readme.md
gruntfile.js
package.json
info:
info: you may need to run `npm install` to install some dependancies
info: run 'npm start' to start your server
```

package.jsonに追加されたパッケージを`npm install`でインストールします。

```bash
$ docker-compose run --rm npm install
```

## サーバーの起動

Docker Composeのserverサービスに定義してあるentrypointの`npm start`からpackage.jsonの`actionhero start`を実行します。

```bash
$ docker-compose run --rm server

> my_actionhero_project@0.0.1 start /app
> actionhero start

info: actionhero >> start
 * Error fetching this hosts external IP address; setting id base to 'actionhero'
2015-06-21 08:38:12 - notice: *** starting actionhero ***
2015-06-21 08:38:12 - warning: running with fakeredis
2015-06-21 08:38:12 - notice: pid: 13
2015-06-21 08:38:12 - notice: server ID: actionhero
2015-06-21 08:38:12 - info: ensuring the existence of the chatRoom: defaultRoom
2015-06-21 08:38:12 - info: ensuring the existence of the chatRoom: anotherRoom
2015-06-21 08:38:12 - notice: starting server: web
2015-06-21 08:38:12 - notice: starting server: websocket
2015-06-21 08:38:14 - notice: environment: development
2015-06-21 08:38:14 - notice: *** Server Started @ 2015-06-21 08:38:14 ***
2015-06-21 08:38:14 - info: actionhero member actionhero has joined the clust
```

サーバーが起動しましたがエラーがでています。[issues](https://github.com/evantahler/actionhero/issues/517)にあるようにこの`Error`は無視できます。[id.js](https://github.com/evantahler/actionhero/blob/master/initializers/id.js)ファイルでクラスタ用の`app.id`にIPアドレスを取得仕様として失敗します。気になったら環境変数の`ACTIONHERO_TITLE`を指定するとよいみたいです。

ブラウザからDockerホストのIPアドレスを指定します。なかなかかっこいいトップページです。

![actionhero.png](/2015/06/20/actionherojs-install/actionhero.png)

