title: "Hapi.jsでHappy Coding - Part2: MongoDBとSwaggerのCRUD"
date: 2015-06-12 20:27:01
tags:
 - Hapijs
 - SocketIO
 - Nodejs
 - Swagger
 - MongoDB
 - Mongoose
 - joi
description: 前回に続いてBuild RESTful API Using Node and Hapiのチュートリアルを読みながらMongoDBを使ったCRUDの実装を勉強していきます。リポジトリはこちらにpushしています。
---

[前回](/2015/06/11/hapijs-swagger-plugin/)に続いて[Build RESTful API Using Node and Hapi](http://www.tothenew.com/blog/build-restful-api-using-node-and-hapi/)のチュートリアルを読みながらMongoDBを使ったCRUDの実装を勉強していきます。リポジトリは[こちら](https://github.com/masato/docker-swagger/tree/mongo)にpushしています。

<!-- more -->

## プロジェクト

前回作成したプロジェクトにmodelを追加しました。

```bash
$ tree
.
├── Dockerfile
├── app.js
├── docker-compose.yml
├── models
│   └── job.js
├── mongo
├── node_modules -> /dist/node_modules
└── package.json
```

### MongoDBサービス

docker-compose.ymlにMongoDBサービスを追加します。データのディレクトリはDockerホストにマウントして開発中は気軽にコンテナの破棄をできるようにしました。

```yaml:docker-compose.yml
hapi:
  build: .
  ports:
    - 3000:3000
  volumes:
    - .:/app
  links:
    - mongo
mongo:
  image: mongo
  volumes:
    - ./mongo:/data/db
```

カレントディレクトリにrootユーザーが所有する`mongo`ディレクトリができるので、Dockerの管理外にします。

```bash:.dockerignore
mongo
```

### Mongooseとjoi

MongoDBにアクセスするために[Mongoose](http://mongoosejs.com/)を使います。MongoDBはスキーマレスですがMongooseはスキーマを定義します。JSONにスキーマを持たせてある程度きっちりレコードを管理したい場合、フォームのバリデーションライブラリの[joi](https://github.com/hapijs/joi)と併せて使うと煩雑になりがちなコードが宣言型ですっきりします。

```js app.js
'use strict';

var Hapi = require('hapi'),
    server = new Hapi.Server(),
    Joi = require('joi'),
    mongoose = require('mongoose');

mongoose.connect('mongodb://'+process.env.MONGO_PORT_27017_TCP_ADDR+':'
              +process.env.MONGO_PORT_27017_TCP_PORT+'/jobdb');

var JobModel = require('./models/job');

server.connection({port: 3000});
```

以下のようなモデルのスキーマを定義します。Hapi.jsは[agenda](https://github.com/rschmukler/agenda)のフロントに使う予定なのでジョブのモデルを作りました。

```js models/job.js
'use strict';

var mongoose = require('mongoose');
var Schema = mongoose.Schema;

var JobSchema = new Schema({
    name: String,
    query: String
});

module.exports = mongoose.model('Job', JobSchema);
```

## CRUD

[LoopBack](http://loopback.io/)の場合ジェネレーターのおかげでコードを書かなくてもCRUDの操作が実装できます。今回のHapi.jsのサンプルはスカッフォールディングで十分なくらい単純ですが、Mongooseとjoiを使ったリクエストパラメーターの扱いを理解すればサクサク実装できそうな気がします。

### POST /api/jobs

POSTメソッドを例にします。`tags: ['api']`はSwagger用に必須項目になります。モデル保存後のコールバックは冗長な感じです。`pyramid of doom`.

```js app.js
server.route({
    method: 'POST',
    path: '/api/jobs',
    config: {
        tags: ['api'],
        description: 'Save job data',
        notes: 'Save job data',
        validate: {
            payload: {
                name: Joi.string().required(),
                query: Joi.string().required()
            }
        }
    },
    handler: function(request, reply) {
        var job = new JobModel(request.payload);
        job.save(function(error) {
            if(error) {
                reply({
                    statusCode: 503,
                    message: error
                });
            } else {
                reply({
                    statusCode: 201,
                    message: 'Job Saved Successfully'
                });
            }
        });
    }
});
```

## Swaggerの起動

Docker Composeからhapiサービスをupします。

```bash
$ docker-compose up hapi
Recreating dockerhapi_mongo_1...
Recreating dockerhapi_hapi_1...
Attaching to dockerhapi_hapi_1
hapi_1 |
hapi_1 | > docker-hapi@0.0.1 start /app
hapi_1 | > node app.js
hapi_1 |
hapi_1 | Server running at: http://22fb2b92f597:3000
```

ブラウザでSwagger UIを表示します。CRUD処理のAPIの動くドキュメントが簡単に作成できました。

![swagger-crud.png](/2015/06/12/hapijs-mongodb/swagger-crud.png)
