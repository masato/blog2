title: "RethinkDB on Docker - Part2: 10 Minutes"
date: 2015-05-26 09:26:45
tags:
 - RethinkDB
 - Docker
 - Node.js
description: 前回少しだけRethinkDBを使ってみてとてもいい感じで使いやすいです。APIリファレンスもNode.js、Python、Rubyとシンプルでみやすく充実しています。The Thinkerのマスコットもかわいいです。Thirty-second quickstart with RethinkDBに続いてチュートリアルのTen-minute guide with RethinkDB and JavaScriptを試してみます。

---

[前回](/2015/05/25/rethinkdb-on-docker/)少しだけRethinkDBを使ってみてとてもいい感じで使いやすいです。APIリファレンスも[Node.js](http://rethinkdb.com/api/javascript/)、[Python](http://rethinkdb.com/api/python/)、[Ruby](http://rethinkdb.com/api/ruby/)とシンプルでみやすく充実しています。[The Thinker](http://www.annieruygtillustration.com/blog/2015/3/12/uktt673n0teivjklu8g4x0bgrl4ymv)のマスコットもかわいいです。[Thirty-second quickstart with RethinkDB](http://www.rethinkdb.com/docs/quickstart/)に続いてチュートリアルの[Ten-minute guide with RethinkDB and JavaScript](http://rethinkdb.com/docs/guide/javascript/)を試してみます。

<!-- more -->

## 10 Minutes

プログラムを理解してしまえば10分で動きますが、Node.jsの動くプログラムを書くにはもう少し時間がかかります。こちらは[Node.js](http://rethinkdb.com/docs/guide/javascript/)の他に、[Python](http://rethinkdb.com/docs/guide/python/)、[Ruby](http://rethinkdb.com/docs/guide/ruby/)用が用意されています。

## プロジェクト

Node.jsのプログラムもDockerで動かすのでプロジェクトを作成します。

```bash
$ cd ~/rethinkdb_apps/node-rethinkdb
$ tree -L 1
.
├── Dockerfile
├── app.js
├── node_modules -> /dist/node_modules
└── package.json
```

Dockerfileは[オフィシャル](nodehttps://registry.hub.docker.com/_/node/)のNode v0.12.3です。

```bash ~/rethinkdb_apps/node-rethinkdb/Dockerfile
FROM node:0.12
MAINTAINER Masato Shimizu <ma6ato@gmail.com>

RUN mkdir -p /app
WORKDIR /app

COPY package.json /app/
RUN mkdir -p /dist/node_modules && ln -s /dist/node_modules /app/node_modules && \
  npm install
COPY . /app
CMD ["npm", "start"]
```

`node_modules`をDockerホスト側でキャッシュするために`/dist/node_modules/`をホスト側にシムリンクを作ります。

```bash
$ cd ~/rethinkdb_apps/node-rethinkdb
$ ln -s /dist/node_modules .
```

package.jsonはRethinkDBと非同期処理のために[async](https://github.com/caolan/async)を使います。

```json ~/rethinkdb_apps/node-rethinkdb/package.json
{
  "name": "node-rethinkdb",
  "description": "Node RethinkDB 10 minutes",
  "version": "0.0.1",
  "private": true,
  "dependencies": {
    "rethinkdb": "^2.0.0-1",
    "async" : "^1.0.0"
  },
  "scripts": {"start": "node app.js"}
}
```

[Node.js](http://rethinkdb.com/docs/guide/javascript/)のサンプルは一連で動くように[async.waterfall](https://github.com/caolan/async#waterfall)でまとめました。Node.jsのデータベースアクセスのサンプルプログラムなのでコールバックは多いですがわかりやすいAPIなので何をしているかすぐわかると思います。

```js ~/rethinkdb_apps/node-rethinkdb/app.js
var r = require('rethinkdb'),
async = require('async');

console.log('=== Open a connection');
r.connect({host: process.env.RDB_PORT_8080_TCP_ADDR,
           port: 28015},
    function(err,conn){
        if (err) throw err;
        async.waterfall([            
            function(callback){
                console.log('=== Drop a table');
                r.db('test').tableDrop('authors').run(conn,callback);
            },
            function(res,callback) {
                console.log('=== Create a table');
                r.db('test').tableCreate('authors').run(conn,callback);
            },
            function(res,callback){
                console.log('=== Insert data');
                r.table('authors').insert([
                    { name: "William Adama", tv_show: "Battlestar Galactica",
                      posts: [
                          {title: "Decommissioning speech", content: "The Cylon War is long over..."},
                          {title: "We are at war", content: "Moments ago, this ship received word..."},
                          {title: "The new Earth", content: "The discoveries of the past few days..."}
                      ]
                    },
                    { name: "Laura Roslin", tv_show: "Battlestar Galactica",
                      posts: [
                          {title: "The oath of office", content: "I, Laura Roslin, ..."},
                          {title: "They look like us", content: "The Cylons have the ability..."}
                      ]
                    },
                    { name: "Jean-Luc Picard", tv_show: "Star Trek TNG",
                      posts: [
                          {title: "Civil rights", content: "There are some words I've known since..."}
                      ]
                    }
                ]).run(conn,callback);
            },
            function(res,callback){
                console.log(JSON.stringify(res, null, 2));
                callback(null);
            },
            function(callback){
                console.log('=== All documents in a table');
                r.table('authors').run(conn,callback);
            },
            function(cursor,callback){
                cursor.toArray(callback);
            },
            function(res,callback) {
                console.log(JSON.stringify(res, null, 2));
                callback(null);
            },
            function(callback){
                console.log('=== Filter documents based on a condition');
                r.table('authors')
                    .filter(r.row('name').eq("William Adama"))
                    .run(conn,callback);
            },
            function(cursor,callback){
                cursor.toArray(callback);
            },
            function(res,callback) {
                console.log(JSON.stringify(res, null, 2));
                callback(null);
            },
            function(callback) {
                console.log('=== Retrieve all authors who have more than two posts');
                r.table('authors')
                    .filter(r.row('posts').count().gt(2))
                    .run(conn,callback);
            },
            function(cursor,callback){
                cursor.toArray(callback);
            },
            function(res,callback) {
                console.log(JSON.stringify(res, null, 2));
                callback(null);
            },
            function(callback) {
                console.log('=== Update documents');
                r.table('authors')
                    .update({type: "fictional"})
                    .run(conn,callback);
            },function(res,callback) {
                console.log(JSON.stringify(res, null, 2));
                callback(null);
            },
            function(callback){
                console.log('=== Update a subset of documents by filtering the table first');
                r.table('authors')
                    .filter(r.row("name").eq("William Adama"))
                    .update({rank: "Admiral"})
                    .run(conn,callback);
            },function(res,callback){
                console.log(JSON.stringify(res, null, 2));
                callback(null);
            },
            function(callback){
                console.log('=== add an additional post');
                r.table('authors')
                    .filter(r.row("name").eq("Jean-Luc Picard"))
                    .update({posts: r.row("posts").append({
                             title: "Shakespeare",
                             content: "What a piece of work is man..."})})
                    .run(conn,callback);
            },function(res,callback){
                console.log(JSON.stringify(res, null, 2));
                callback(null);
            },
            function(callback){
                console.log('=== Delete documents');
                r.table('authors')
                    .filter(r.row('posts').count().lt(3))
                    .delete()
                    .run(conn,callback);
            },function(res,callback){
                console.log(JSON.stringify(res, null, 2));
                callback(null);
            },
            function(callback){
                console.log('=== Close a connection');
                conn.close(callback);                
            }
        ],function(err, result){
            if (err) throw err;
        });
});
```


[Realtime feeds](http://rethinkdb.com/docs/guide/javascript/#realtime-feeds)はバッチで実行できないので除外しました。レコードに更新があったときにプッシュ通知してくれるようです。


```js
r.table('authors').changes().run(connection, function(err, cursor) {
    if (err) throw err;
    cursor.each(function(err, row) {
        if (err) throw err;
        console.log(JSON.stringify(row, null, 2));
    });
});
```


### 実行

作成したサンプルプログラムを実行します。Dockerのコンテナはlinkを使いRethinkDBのIPアドレスを環境変数から取得します。

```bash
$ cd ~/rethinkdb_apps/node-rethinkdb
$ docker build -t node-rethinkdb .
$ docker run --rm --name node-rethinkdb --link rethinkdb:rdb -v $PWD:/app node-rethinkdb

> node-rethinkdb@0.0.1 start /app
> node app.js
```

処理結果です。レコードの取得系ははJSONでレコードが返ります。登録、更新、削除系でもJSONで処理結果が返ります。

```json
=== Open a connection
=== Drop a table
=== Create a table
=== Insert data
{
  "deleted": 0,
  "errors": 0,
  "generated_keys": [
    "f96b4b0f-5e5a-4484-a973-839038ae9725",
    "c398f4a0-bfb7-42c0-92d0-54830ee74460",
    "63137b0a-4d55-4635-8fd7-1bb1120ab8bd"
  ],
  "inserted": 3,
  "replaced": 0,
  "skipped": 0,
  "unchanged": 0
}
=== All documents in a table
[
  {
    "id": "f96b4b0f-5e5a-4484-a973-839038ae9725",
    "name": "William Adama",
    "posts": [
      {
        "content": "The Cylon War is long over...",
        "title": "Decommissioning speech"
      },
        "content": "I, Laura Roslin, ...",
        "title": "The oath of office"
      },
      {
        "content": "The Cylons have the ability...",
        "title": "They look like us"
      }
    ],
    "tv_show": "Battlestar Galactica"
  },
  {
    "id": "63137b0a-4d55-4635-8fd7-1bb1120ab8bd",
    "name": "Jean-Luc Picard",
    "posts": [
      {
        "content": "There are some words I've known since...",
        "title": "Civil rights"
      }
    ],
    "tv_show": "Star Trek TNG"
  }
]
=== Filter documents based on a condition
[
  {
    "id": "f96b4b0f-5e5a-4484-a973-839038ae9725",
    "name": "William Adama",
    "posts": [
      {
        "content": "The Cylon War is long over...",
        "title": "Decommissioning speech"
      },
      {
        "content": "Moments ago, this ship received word...",
        "title": "We are at war"
      },
      {
        "content": "The discoveries of the past few days...",
        "title": "The new Earth"
      }
    ],
    "tv_show": "Battlestar Galactica"
  }
]
=== Retrieve all authors who have more than two posts
[
  {
    "id": "f96b4b0f-5e5a-4484-a973-839038ae9725",
    "name": "William Adama",
    "posts": [
      {
        "content": "The Cylon War is long over...",
        "title": "Decommissioning speech"
      },
      {
        "content": "Moments ago, this ship received word...",
        "title": "We are at war"
      },
      {
        "content": "The discoveries of the past few days...",
        "title": "The new Earth"
      }
    ],
    "tv_show": "Battlestar Galactica"
  }
]
=== Update documents
{
  "deleted": 0,
  "errors": 0,
  "inserted": 0,
  "replaced": 3,
  "skipped": 0,
  "unchanged": 0
}
=== Update a subset of documents by filtering the table first
{
  "deleted": 0,
  "errors": 0,
  "inserted": 0,
  "replaced": 1,
  "skipped": 0,
  "unchanged": 0
}
=== add an additional post
{
  "deleted": 0,
  "errors": 0,
  "inserted": 0,
  "replaced": 1,
  "skipped": 0,
  "unchanged": 0
}
=== Delete documents
{
  "deleted": 2,
  "errors": 0,
  "inserted": 0,
  "replaced": 0,
  "skipped": 0,
  "unchanged": 0
}
=== Close a connection
```
