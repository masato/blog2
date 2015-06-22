title: "actionhero.js入門 - Part2: チュートリアル (アクションの作成)"
date: 2015-06-21 21:41:14
tags:
 - actionherojs
 - Nodejs
 - API
 - Redis
 - async
 - DockerCompose
description: Getting Startedで作成したプロジェクトにactionhero-tutorialを読みながらチュートリアルを実行していきます。写経していたらチュートリアルの翻訳みたいになりました。技術翻訳は楽しい作業なので気分転換になります。
---

[Getting Started](/2015/06/20/actionherojs-install/)で作成したプロジェクトに[actionhero-tutorial](https://github.com/evantahler/actionhero-tutorial)を読みながらチュートリアルを実行していきます。写経していたらチュートリアルの翻訳みたいになりました。技術翻訳は楽しい作業なので気分転換になります。


<!-- more -->

## プロジェクトの準備

今回の作業はこちらの[リポジトリ](https://github.com/masato/docker-actionhero/tree/actions)にpushしています。

### プロジェクト

[前回](/2015/06/20/actionherojs-install/)作成したプロジェクトに移動します。

```bash
$ cd ~/node_apps/docker_actionhero
```

雛形のpackage.jsonをblog用に編集します。


```json ~/node_apps/docker_actionhero/package.json
 {
  "author"      : "Masato Shimizu <ma6ato@gmail.com>",
  "name"        : "my-blog",
  "description" : "my blog project",
  "version"     : "0.0.1",
...
```

プロジェクトをここでコミットしておきます。

```bash
$ git init
$ git add -A
$ git commit -m 'first commit'
```

### チュートリアルのRedisとasync

[Getting Started](/2015/06/20/actionherojs-install/)の[リポジトリ](https://github.com/evantahler/actionhero-tutorial/)とジェネレートされるコードに差分がありますが、ジェネレートに自分で写経したコードを使っていきます。

actionheroのコードは[LoopBack](http://loopback.io/)のようなオートマジックはないので結構がりがりと書きますが、Express
でスクラッチから書くよりは安全です。Redisをデータストアに使うサンプルとしても勉強になります。asyncもRedisもちょっとしたデータモデルを設計するとすぐに複雑になります。

またイニシャライザやアクションのコードは[async](https://github.com/caolan/async)で書きます。チュートリアルasyncの実践にもなります。

### Redisのリンク


アプリの構成管理はDocker Composeを使います。serverサービスでサーバーを起動します。

```yaml ~/node_apps/docker-actionhero/docker-compose.yml
server:  &defaults
  image: masato/actionhero
  volumes:
    - .:/app
    - /etc/localtime:/etc/localtime:ro
  environment:
    - REDIS_HOST=redisdb
  ports:
    - 8089:8080
  links:
    - redis:redisdb
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
    - /etc/localtime:/etc/localtime:ro
rediscli:
  image: redis
  links:
    - redis
```

serverサービスとredisサービスのリンクしているので、environmentに`redis`を指定しても動作しそうに見えます。

```yaml
  environment:
    - REDIS_HOST=redis
  links:
    - redis
```

なぜか以下のようなエラーが発生してしまいます。

```bash
server_1 | 2015-06-22 20:45:03 - emerg: Redis Error (client): Error: Redis connection to tcp://172.17.6.22:6379 failed - connect ENOENT
```

サービス名にエイリアスを作成するとRedisに接続できるようになりました。`REDIS_*`の環境変数が何か問題を起こしているようです。

```yaml
  environment:
    - REDIS_HOST=redisdb
  ports:
    - 8089:8080
  links:
    - redis:redisdb
```

### docker-composeのエイリアスの作成

Docker Composeを使いワンショットでサービスを実行するコマンドがノイズになってきたのでエイリアスを書きます。

```bash ~/.bashrc
alias actionhero='docker-compose run --rm actionhero'
```

`~/.bashrc`を再読込して`actionhero`のエイリアスをテストします。

```bash
$ source ~/.bashrc
$ actionhero help
info: actionhero >> help
info: actionhero - a node.js API framework for both tcp sockets, web sockets, and http clients.

Binary options:
* help (default)
* start
* startCluster
* generate
* generateAction
* generateTask
* generateInitializer
* generateServer
...
```

## イニシャライザ

### blog

このセクションでは[initializers/blog.js](https://github.com/evantahler/actionhero-tutorial/blob/master/initializers/blog.js)を作成します。ドキュメントは[Initializers](http://actionherojs.com/docs/core/initializers.html)を参照します。

イニシャライザにはアプリの共通コードを書きます。データベースに接続するモデルやミドルウェアなどです。ここで作成したクラスはapiオブジェクトに追加して使います。例をあげると`api.mysql`や`api.game`はアクションやタスクのスコープで利用することができます。別の方法ではサーバーを起動したとき`_start`メソッド内でコードを実行することもできます。

今回はblogを構築しているので、最初に投稿やコメントの保存場所が必要です。actionheroでは最初からRedisが`api.redis.client`を通して使えるのでさっそくデータを保存してみます。`blog`イニシャライザを新規作成します。

```bash
$ cd ~/node_apps/docker-actionhero
$ actionhero generateInitializer --name=blog
info: actionhero >> generateInitializer
info:  - wrote file '/app/initializers/blog.js'
```

`/app/initializers/blog.js`にblogの共通関数を定義します。

```js ~/node_apps/docker-actionhero/initializers/blog.js
'use strict';

module.exports = {
    loadPriority:  1000,
    startPriority: 1000,
    stopPriority:  1000,

    initialize: function(api, next){
        var redis = api.redis.client;
        api.blog = {

            // constants
            separator: ";",
            postPrefix: "posts",
            commentPrefix: "comments:",

            // posts
            postAdd: function(userName, title, content, next){
                var key = this.buildTitleKey(userName, title);
                var data = {
                    content: content,
                    title: title,
                    userName: userName,
                    createdAt: new Date().getTime(),
                    updatedAt: new Date().getTime(),
                };
                redis.hmset(key, data, function(error){
                    next(error);
                });
            },
...            
}
```

コードの補足説明です。

* `posts`はRedisのHash型です。contentといくつかのmeta dataを持ちます。
* `comments`もRedisのHash型です。comment毎のkeyを持ちます。
* すべて非同期関数を作成します。asyncの慣例として常に`callback(error,data0)を返します。
* このレイヤーでは認証やバリデーションは気にしません。


### ユーザーと認証

このセクションでは[initializers/users.js](https://github.com/evantahler/actionhero-tutorial/blob/master/initializers/users.js)と[package.json](https://github.com/evantahler/actionhero-tutorial/blob/master/package.json)を作成します。

blogは通常ユーザー認証が必要です。先ほどと同様にコマンドを実行してinitializerを作成します。

```bash
$ actionhero generateInitializer --name=users
info: actionhero >> generateInitializer
info:  - wrote file '/app/initializers/users.js'
```

`initializers/users.js`は以下のように作成されました。

```js ~/node_apps/docker-actionhero/initializers/users.js
'use strict';

var crypto = require('crypto');
var salt = "asdjkafhjewiovnjksdv";

module.exports = {
    loadPriority:  1000,
    startPriority: 1000,
    stopPriority:  1000,

    initialize: function(api, next){
        var redis = api.redis.client;
        api.users = {
            // constants
            usersHash: "users",
            
            // methods
            add: function(userName, password, next){
                var self = this;
                redis.hget(self.usersHash, userName, function(error, data){
                    if(error){
                        next(error);
                    }else if(data){
                        next("userName already exists");
                    }else{
                        self.cryptPassword(password, function(error, hashedPassword){
                            if(error){
                                next(error);
                            }else{
                                var data = {
                                    userName: userName,
                                    hashedPassword: hashedPassword,
                                    createdAt: new Date().getTime(),
                                };
                                redis.hset(self.usersHash, userName, JSON.stringify(data), function(error){
                                    next(error);
                                });
                            }
                        });
                    }
                });
            },
...
}
```

いくつか注意事項です。

* 先ほどと同様にデータは全てRedisのHash型に保存します。
* `user`を削除したときは関連する`posts`と`comments`も削除する必要があります。
* md5だけを使って`user`のパスワードをハッシュ化します。プロダクションではよりセキュアな方法を選択すべきで、たとえば[BCrypt](https://github.com/ncb000gt/node.bcrypt.js/)などを使います。


### ミドルウェアのpublicとprivateなアクション

次に[initializers/middleware.js](https://github.com/evantahler/actionhero-tutorial/blob/master/initializers/middleware.js)を作成します。ドキュメントは[Middleware](http://actionherojs.com/docs/core/middleware.html)です。

上記のステップで`api.users.authenticate`メソッドを作成しましたがまだ使っていません。このメソッドは明らかに保護が必要なメソッドです。ポストを追加したりユーザーを削除したりするメソッドも同様です。ここには何らかのセーフガードを導入する必要があります。


actionheroではラップして`users`の全てのイニシャライザのメソッドはアクション内で利用します。さっそくミドルウェアを作成してアクションに追加してみます。

```bash
$ actionhero generateInitializer --name=middleware
info: actionhero >> generateInitializer
info:  - wrote file '/app/initializers/middleware.js'
```

actionheroでは関数の配列をアクションの前後で実行することができます。今回必要なのは事前チェックでアクションを実行してよいか判断する関数です。この関数からアクション自体にアクセスもできます。データベースのコネクションも使えます。ミドルウェアは`authenticated = true`をアクションの定義に追加すると有効になって実行されます。ミドルウェア側では`api.actions.addPreProcessor(authenticationMiddleware)`のようにしてアクションに追加します。


## アクションの作成

このセクションでは[actions/users.js](https://github.com/evantahler/actionhero-tutorial/blob/master/actions/users.js)と[actions/blog.js](https://github.com/evantahler/actionhero-tutorial/blob/master/actions/blog.js)を作成します。ドキュメントは[Actions](http://actionherojs.com/docs/core/actions.html)です。

`posts`イニシャライザにgetter/setterなどヘルパー関数を定義しました。この関数はアクション内で使います。1つのファイルに複数のアクションを定義します。`comments`を使うアクションと`posts`を使うアクションを作成します。

```bash
$ actionhero generateAction --name=users
$ actionhero generateAction --name=blog
```

アクションに`authenticated = true`を追加して必要なセキュリティ機能を有効にします。

## テスト

Docker Composeからserverサービスをupします。

```bash
$ docker-compose up server
Recreating dockeractionhero_redis_1...
Creating dockeractionhero_server_1...
Attaching to dockeractionhero_server_1
server_1 |
server_1 | > my-blog@0.0.1 start /app
server_1 | > actionhero start
server_1 |
server_1 | info: actionhero >> start
server_1 | 2015-06-22 21:42:31 - notice: *** starting actionhero ***
server_1 | 2015-06-22 21:42:31 - notice: pid: 15
server_1 | 2015-06-22 21:42:31 - notice: server ID: 172.17.6.72
server_1 | 2015-06-22 21:42:31 - info: ensuring the existence of the chatRoom: defaultRoom
server_1 | 2015-06-22 21:42:31 - info: ensuring the existence of the chatRoom: anotherRoom
server_1 | 2015-06-22 21:42:31 - info: actionhero member 172.17.6.72 has joined the cluster
server_1 | 2015-06-22 21:42:31 - notice: starting server: web
server_1 | 2015-06-22 21:42:31 - notice: starting server: websocket
server_1 | 2015-06-22 21:42:32 - notice: environment: development
server_1 | 2015-06-22 21:42:32 - notice: *** Server Started @ 2015-06-22 21:42:32 ***
```

### ユーザーの追加

ユーザーの登録をします。

```bash
$ curl -X POST \
  -d "userName=evan" \
  -d "password=password" \
  "http://localhost:8089/api/userAdd"
```

以下のレスポンスが返ります。

```json
{
  "serverInformation": {
    "serverName": "actionhero API",
    "apiVersion": "0.0.1",
    "requestDuration": 9,
    "currentTime": 1434977027506
  },
  "requesterInformation": {
    "id": "137e30e7478dac9cd57c13b4bc91a62eb1b14d2b-b67044b1-93fc-4541-9f87-8299f1b8704a",
    "fingerprint": "137e30e7478dac9cd57c13b4bc91a62eb1b14d2b",
    "remoteIP": "172.17.42.1",
    "receivedParams": {
      "userName": "evan",
      "password": "password",
      "action": "userAdd",
      "apiVersion": 1
    }
  }
}
```

ログは以下のように出力されます。

```bash
server_1 | 2015-06-22 21:43:47 - info: [ action @ web ] to=172.17.42.1, action=userAdd, params={"userName":"evan","password":"password","action":"userAdd","apiVersion":1}, duration=3
```

Redisのレコードを確認してみます。Hash型のusersにレコードが作成されました。

```bash
$ docker-compose run --rm  rediscli bash -c 'redis-cli -h $REDIS_PORT_6379_TCP_ADDR'
172.17.6.28:6379> keys "*"
172.17.6.71:6379> keys "*"
1) "actionhero:chatRoom:rooms"
2) "actionhero:stats"
3) "users"
172.17.6.71:6379> type "users"
hash
172.17.6.71:6379> hgetall "users"
1) "evan"
2) "{\"userName\":\"evan\",\"hashedPassword\":\"8c0efc73baa95ec0b43f2f9c6515e3a7\",\"createdAt\":1434977027504}"
```

### ログイン

ログインをしてみます。

```bash
$ curl -X POST \
  -d "userName=evan" \
  -d "password=password" \
  "http://localhost:8089/api/authenticate"
```

以下のレスポンスが返ります。`"authenticated": true`が出力されて成功しました。

```json
{
  "authenticated": true,
  "serverInformation": {
    "serverName": "actionhero API",
    "apiVersion": "0.0.1",
    "requestDuration": 5,
    "currentTime": 1434977341898
  },
  "requesterInformation": {
    "id": "98a942efd4527a03678ce104adc6d9d9d89cd35b-018bde25-ebfa-420a-8c68-714f3749a473",
    "fingerprint": "98a942efd4527a03678ce104adc6d9d9d89cd35b",
    "remoteIP": "172.17.42.1",
    "receivedParams": {
      "userName": "evan",
      "password": "password",
      "action": "authenticate",
      "apiVersion": 1
    }
  }
}
```

### 投稿する

blogに投稿してみます。

```bash
$ curl -X POST \
  -d "userName=evan" \
  -d "password=password" \
  -d "title=first-post" \
  -d "content=My%20first%20post.%20%20Yay." \
  "http://localhost:8089/api/postAdd"
```

以下のレスポンスが返ります。特に成功を示す値は返らないようです。エラーになっていないので成功しているようです。

```json
{
  "serverInformation": {
    "serverName": "actionhero API",
    "apiVersion": "0.0.1",
    "requestDuration": 3,
    "currentTime": 1434978106847
  },
  "requesterInformation": {
    "id": "e66f4f3ff7b52631513053f2c93c268153d0f591-315a8670-8dc4-4101-b520-81501da95e48",
    "fingerprint": "e66f4f3ff7b52631513053f2c93c268153d0f591",
    "remoteIP": "172.17.42.1",
    "receivedParams": {
      "userName": "evan",
      "password": "password",
      "title": "first-post",
      "content": "My first post.  Yay.",
      "action": "postAdd",
      "apiVersion": 1
    }
  }
```

Redisには`"posts;evan;first-post"`のキーでHash型のレコードが作成されました。

```
172.17.6.71:6379> keys "*"
1) "actionhero:chatRoom:rooms"
2) "posts;evan;first-post"
3) "actionhero:stats"
4) "users"
172.17.6.71:6379> type "posts;evan;first-post"
hash
172.17.6.71:6379> hgetall "posts;evan;first-post"
 1) "content"
 2) "My first post.  Yay."
 3) "title"
 4) "first-post"
 5) "userName"
 6) "evan"
 7) "createdAt"
 8) "1434978106846"
 9) "updatedAt"
10) "1434978106846"
```

### 投稿を取得する

認証があるので投稿はPOSTメソッドで取得します。

```bash
$ curl -X POST \
  -d "userName=evan" \
  -d "title=first-post" \
  "http://localhost:8089/api/postView"
```

レスポンスは以下です。

```json
{
  "post": {
    "content": "My first post.  Yay.",
    "title": "first-post",
    "userName": "evan",
    "createdAt": "1434978106846",
    "updatedAt": "1434978106846"
  },
  "serverInformation": {
    "serverName": "actionhero API",
    "apiVersion": "0.0.1",
    "requestDuration": 2,
    "currentTime": 1434978387670
  },
  "requesterInformation": {
    "id": "138ec5166fe759881affdc39c99e3cfde8164898-86704c11-8001-41c6-badc-83a06243ee5d",
    "fingerprint": "138ec5166fe759881affdc39c99e3cfde8164898",
    "remoteIP": "172.17.42.1",
    "receivedParams": {
      "userName": "evan",
      "title": "first-post",
      "action": "postView",
      "apiVersion": 1
    }
  }
}
```

### コメントを追加する

```bash
$ curl -X POST \
  -d "userName=evan" \
  -d "title=first-post" \
  -d "comment=cool%20post" \
  -d "commenterName=someoneElse" \
  "http://localhost:8089/api/commentAdd"
```

以下のレスポンスが返ります。

```json
{
  "serverInformation": {
    "serverName": "actionhero API",
    "apiVersion": "0.0.1",
    "requestDuration": 2,
    "currentTime": 1434978527198
  },
  "requesterInformation": {
    "id": "854e96b72fa0341efb5b408af852a6d210a83e31-0466bf62-a83b-4f9a-b431-b7543baa6f05",
    "fingerprint": "854e96b72fa0341efb5b408af852a6d210a83e31",
    "remoteIP": "172.17.42.1",
    "receivedParams": {
      "userName": "evan",
      "title": "first-post",
      "comment": "cool post",
      "commenterName": "someoneElse",
      "action": "commentAdd",
      "apiVersion": 1
    }
  }
}
```

Redisに`"comments:;evan;first-post"`のキーでHash型のレコードが作成されました。コメントはJSONの文字列が保存されています。

```bash
172.17.6.71:6379> keys "*"
1) "actionhero:stats"
2) "users"
3) "actionhero:chatRoom:rooms"
4) "posts;evan;first-post"
5) "comments:;evan;first-post"
172.17.6.71:6379> type "comments:;evan;first-post"
hash
172.17.6.71:6379> hgetall "comments:;evan;first-post"
1) "someoneElse1434978527197"
2) "{\"comment\":\"cool post\",\"createdAt\":1434978527197,\"commentId\":\"someoneElse1434978527197\"}"
```

### コメントを見る

コメントも認証が入るのでPOSTで取得します。

```bash
$ curl -X POST \
  -d "userName=evan" \
  -d "title=first-post" \
  "http://localhost:8089/api/commentsView"
```

レスポンスのJSONからコメントを取得するときは、`comments`のキーで取得します。

```json
{
  "comments": [
    {
      "comment": "cool post",
      "createdAt": 1434978527197,
      "commentId": "someoneElse1434978527197"
    }
  ],
  "serverInformation": {
    "serverName": "actionhero API",
    "apiVersion": "0.0.1",
    "requestDuration": 2,
    "currentTime": 1434978785461
  },
  "requesterInformation": {
    "id": "9fff33d8178275b64e168c1cc26bf7e1e9fb3bd8-df377114-716b-4558-9dc0-385f40a7efb4",
    "fingerprint": "9fff33d8178275b64e168c1cc26bf7e1e9fb3bd8",
    "remoteIP": "172.17.42.1",
    "receivedParams": {
      "userName": "evan",
      "title": "first-post",
      "action": "commentsView",
      "apiVersion": 1
    }
  }
}
```

テストも通ったのでコミットしておきます。

``` bash
$ git add .
$ git commit -m 'actions created'
```