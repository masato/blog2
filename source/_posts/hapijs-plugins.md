title: "Hapi.jsでHappy Coding - Part3: プラグインをつくる"
date: 2015-06-13 21:24:32
tags:
 - Hapijs
 - Nodejs
 - Swagger
description: MongoDBへのCRUD操作とSwagger UIからAPIをテストするプロジェクトをHapi.jsで作成しました。app.jsのRouteの処理が長くなったので、別ファイルに移動しようと思います。Hapi.jsではPluginsの仕組みで機能拡張ができます。Swaggerのインタフェースを提供するhapi-swaggerはnpmからパッケージをインストールしましたが、プラグインはアプリ内にも簡単に作ることができるのでRoute処理をプラグインにリファクタリングします。
---

MongoDBへのCRUD操作とSwagger UIからAPIをテストする[プロジェクト](/2015/06/12/hapijs-mongodb/)を[Hapi.js](http://hapijs.com/)で作成しました。app.jsのRouteの処理が長くなったので、別ファイルに移動しようと思います。Hapi.jsでは[Plugins](http://hapijs.com/tutorials/plugins)の仕組みで機能拡張ができます。[Swagger](http://swagger.io/)のインタフェースを提供する[hapi-swagger](https://www.npmjs.com/package/hapi-swagger)はnpmからパッケージをインストールしましたが、プラグインはアプリ内にも簡単に作ることができるのでRoute処理をプラグインにリファクタリングします。

<!-- more -->

公開されているプラグインはこちらの[ページ](http://hapijs.com/plugins)に用途別にまとまっています。

## プロジェクト

Routeのプラグインを作成したリポジトリは[こちら](https://github.com/masato/docker-swagger/tree/plugin)です。ディレクトリは以下のようになっています。

```bash
$ cd ~/node_apps/docker-hapi
$ tree -L 2
.
├── Dockerfile
├── app.js
├── docker-compose.yml
├── models
│   └── job.js
├── mongo
├── node_modules -> /dist/node_modules
├── npm-debug.log
├── package.json
└── routes
    └── jobs.js
```

今までと同様にDocker Composeを使いMongoDBとlinkしています。

```yaml ~/node_apps/docker-hapi/docker-compose.yml
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

## routes/jobs.js

`routes/jobs.js`が作成したプラグインです。長くなるのでPOST以外は省略します。今までapp.jsに定義していたRouteの処理を`exports.register`の関数内に移動しました。

```js ~/node_apps/docker-hapi/routes/jobs.js
'use strict';
var Joi = require('joi');
var JobModel = require('../models/job');

exports.register = function(server, options, next) {

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
...
    // callback to complete
    next();
}

exports.register.attributes = {
    name: 'jobs-route'
}
```

重要なのは関数の最後にコールバックの`next()`を実行してこのプラグインでの処理の終了を伝えることです。また`exports.register.attributes`にこのプラグインの名前を定義しておきます。

## app.js

Routeの処理を別ファイルにプラグインとして移動したのでapp.jsの構成はすっきりと短くなりました。プラグインの登録は`server.register`を実行して行います。Swaggerのプラグインと加えて2つになったので`plugins`配列に格納します。

```js ~/node_apps/docker-hapi/routes/app.js
'use strict';

var Hapi = require('hapi'),
    server = new Hapi.Server(),
    mongoose = require('mongoose');

mongoose.connect('mongodb://'+process.env.MONGO_PORT_27017_TCP_ADDR+':'
              +process.env.MONGO_PORT_27017_TCP_PORT+'/jobdb');
server.connection({port: 3000});

var plugins = [
    { register: require('hapi-swagger'),
      options: {
          apiVersion: "0.0.1"
      }
    },
    { register: require('./routes/jobs')}
];

server.register(plugins, function(err){
    if (err) throw err;
    server.start(function (){
        console.log('Server running at:', server.info.uri);
    });
});
```

## プログラムの実行

Docker Composeからhapiサービスを実行します。

``` bash
$ docker-compose up hapi
Recreating dockerhapi_mongo_1...
Recreating dockerhapi_hapi_1...
Attaching to dockerhapi_hapi_1
hapi_1 |
hapi_1 | > docker-hapi@0.0.1 start /app
hapi_1 | > node app.js
hapi_1 |
hapi_1 | Server running at: http://803cef3ac3ca:3000
```

Routeをプラグイン化してもSwaggerUIからMongoDBへのCRUD処理はできるのでリファクタリングは成功です。

![hapi-plugin.png](/2015/06/13/hapijs-plugins/hapi-plugin.png)

