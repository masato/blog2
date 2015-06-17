title: "Hapi.jsでHappy Coding - Part1: ルーティングとSwaggerプラグイン"
date: 2015-06-11 22:07:33
tags:
 - Hapijs
 - SocketIO
 - Nodejs
 - Swagger
description: Node.jsでREST APIを作成するフレームワークを調べていますがHapi.jsが良い感じです。いくつかチュートリアルで基本的な操作を勉強していきます。プラグインで機能を拡張することができるためベースはとても小さくできています。Swaggerのプラグインがあったので使ってみます。ただしLoopBackと違いメタデータを自分で定義する必要があります。
---

Node.jsでREST APIを作成するフレームワークを調べていますがHapi.jsが良い感じです。いくつかチュートリアルで基本的な操作を勉強していきます。プラグインで機能を拡張することができるためベースはとても小さくできています。Swaggerのプラグインがあったので使ってみます。ただし[LoopBack](/2015/06/09/nodejs-loopback-on-docker-compose-mongodb/)と違いメタデータを自分で定義する必要があります。

<!-- more -->

## プロジェクト

今回作成するプロジェクトのディレクトリ構成です。ソースコードは[リポジトリ](https://github.com/masato/docker-swagger/tree/swagger)に`swagger`タグでpushしています。

```bash
$ cd ~/node_apps/docker-hapi
$ tree -L 1
.
├── Dockerfile
├── app.js
├── docker-compose.yml
├── mongo
├── node_modules -> /dist/node_modules
├── npm-debug.log
└── package.json
```

docker-compose.ymlのmongoサービスでdataディレクトリをカレントにマウントしています。mongoディレクトリの所有はrootになるのでbuild時にエラーになります。MongoDBのデータディレクトリをDockerイメージに追加しないように、.dockerignoreに追加して無視します。

```bash ~/node_apps/docker-hapi/.dockerignore
mongo
```

今回使うHapi.jsのプラグインの[hapi-swagger](https://github.com/glennjones/hapi-swagger/)がNode.jsのバージョンに[0.10.x](https://github.com/glennjones/hapi-swagger/blob/master/package.json)を指定しているためベースイメージもこれにあわせます。

```package.json
  "engines": {
    "node": "0.10.x"
  },
```

これからこのプロジェクトでHapi.jsのプラグインとMongoDBを使ったサンプルを書いていこうと思います。

```json:~/node_apps/docker-hapi/package.json
{
  "name": "docker-hapi",
  "description": "node-static app",
  "version": "0.0.1",
  "private": true,
  "dependencies": {
      "hapi": "^8.6.1",
      "hapi-swagger": "^0.7.3",
      "good-console": "^5.0.2",
      "joi": "^6.4.3",
      "mongoose": "^4.0.4"
  },
  "scripts": {"start": "node app.js"}
}
```

## ルーティング

SinatraやExpressと同じようにルーティングはHTTPメソッドとURLのパターンで構成します。

### Hello World

やはり最初は公式の[Getting Started](http://hapijs.com/tutorials)から始めます。写経して感じをつかむのには丁度良い長さです。定番のHello Worldをapp.jsに実装します。

```js ~/node_apps/docker-hapi/app.js
'user strict';

var Hapi = require('hapi'),
    server = new Hapi.Server();

server.connection({port: 3000});

server.route({
    method: 'GET',
    path: '/{name}',
    handler: function(request, reply) {
        reply('Hello, ' + encodeURIComponent(request.params.name));
    }
});

server.start(function() {
    console.log('Server running at:', server.info.uri);
});
```

app.jsにパラメータ付きのpathでGETメソッドのRouteを1つ定義しています。URLのパラメーターはpathに定義したnameにバインドされて、`request.params.name`の変数に格納されます。

```bash
$ curl -X GET localhost:3000/masato
Hello, masato
```

### JSON

次は[Build RESTful API Using Node and Hapi](http://www.tothenew.com/blog/build-restful-api-using-node-and-hapi/)を参考にしながらHapi.jsの使い方を勉強していきます。

app.jsに静的なJSONを返すRouteを追加します。APIの追加はmethod、path、handlerを追加していくのが基本的なコードになります。

```js ~/node_apps/docker-hapi/app.js
....
server.route({
    method: 'GET',
    path: '/api/jobs',
    handler: function(request, reply) {
        reply({
            statusCode: 200,
            message: 'List all jobs',
            data: [
                {
                    name: 'Daily BABYMETAL',
                    query: 'babymetal'
                },{
                    nama: 'Hourly Android',
                    query: 'android'
                }
            ]
        });
    }
});
...
```

作成したRouteのpathをcurlでGETします。handlerでreplyしたJSONが返ります。

```bash
$ curl -X GET localhost:3000/api/jobs
{"statusCode":200,"message":"List all jobs","data":[{"name":"Daily BABYMETAL","query":"babymetal"},{"nama":"Hourly Andrond","query":"android"}]}
```

## プラグイン

### Swagger 

Hapi.jsは[プラグイン](http://hapijs.com/plugins)を追加することで認証やAPIドキュメントなどの機能を追加していくことができます。
registerメソッドを使ってプラグインを追加します。

```js ~/node_apps/docker-hapi/app.js
...
server.register({
    register: require('hapi-swagger'),
    options: {
        apiVersion: "0.0.1"
    }
}, function(err){
    if(err) {
        server.log(['error'], 'hapi-swagger load error: ' + err)
    } else {
        server.log(['start'], 'hapi-swagger interface loaded')
    }
});
...
```

SwaggerでAPIドキュメントを作成するため、RouteにSwagger用のconfigを追加します。tagsは必須項目になっています。

```js ~/node_apps/docker-hapi/app.js
...
server.route({
    method: 'GET',
    path: '/api/jobs',
    config: {
        tags: ['api'],
        description: 'List all jobs',
        notes: 'List all jobs'
    },
...
```

Docker Composeをupします。

```bash
$ docker-compose up
```

`/documentation`をブラウザで開くとSwagger UIが使えるようになります。

http://xxx.xxx.xxx:3000/documentation

![swagger.png](/2015/06/11/hapijs-swagger-plugin/swagger.png)

