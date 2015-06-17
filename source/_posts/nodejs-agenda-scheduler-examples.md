title: "Node.jsのagendaでジョブをスケジュールする - Part1: ジョブのCRUD"
date: 2015-06-06 22:06:12
tags:
 - Nodejs
 - agenda
 - MongoDB
 - Schedule
description: 前回リストしたNode.jsのJobqueueとSccheduleツールからスケジュール型ジョブ管理ツールのagendaを使ってみます。別途REST APIを用意してインタフェースする予定なのでジョブの基本的なCRUDを試してみます。agendaはジョブ管理のバックエンドにMongoDBを使います。
---

前回リストした[Node.jsのJobqueueとSccheduleツール](/2015/06/05/nodejs-job-queue-schedule/)からスケジュール型ジョブ管理ツールの[agenda](https://github.com/rschmukler/agenda)を使ってみます。別途REST APIを用意してインタフェースする予定なのでジョブの基本的なCRUDを試してみます。agendaはジョブ管理のバックエンドにMongoDBを使います。

<!-- more -->

## プロジェクトの作成

以下のようなプロジェクトを作成します。リポジトリは[こちら](https://github.com/masato/docker-agenda-sample)です。

``` bash
$ cd ~/node_apps/docker-agenda
$ tree -L 1
.
├── Dockerfile
├── app.js
├── docker-compose.yml
├── node_modules -> /dist/node_modules
└── package.json

1 directory, 5 files
```

app.jsを例にしてagendaのジョブの管理メソッドを確認していきます。

```js ~/node_apps/docker-agenda/app.js 
"use strict";

var Agenda = require('agenda'),
    agenda = new Agenda(
        {db: {address: process.env.MONGO_PORT_27017_TCP_ADDR+':'
              +process.env.MONGO_PORT_27017_TCP_PORT + '/agendadb'}});

agenda.define('hello world', function(job, done) {
  console.log(job.attrs.data.time, 'hello world!');
  done();
});

agenda.schedule('in 5 seconds', 'hello world', {time: new Date()});
agenda.start();
```

## ジョブの作成

### ジョブの処理関数の定義 - define

[define(jobName, [options], fn)](https://github.com/rschmukler/agenda#definejobname-options-fn)はジョブの処理を実装した関数に名前をつけて定義します。関数内では非同期処理の場合はdone()を実行します。同期処理の場合はdoneは省略できます。

```js
agenda.define('hello world', function(job, done) {
  console.log(job.attrs.data.time, 'hello world!');
  done();
});
```

### 1回だけスケジュール - schedule

[schedule(when, name, data)](https://github.com/rschmukler/agenda#schedulewhen-name-data)は`name`のジョブを一度だけ実行します。非英語圏だと前置詞が難しいのですが`in 5 seconds`と書くと`あと5秒経ったら`ジョブを実行します。

```js
agenda.schedule('in 5 seconds', 'hello world', {time: new Date()});
agenda.start();
console.log(new Date(), 'say hello world in 5 seconds');
```

プログラムを実行すると最初のログが表示されてから5秒後に`hello world`が出力されます。

``` bash
$ docker-compose run --rm agenda
Creating dockeragenda_mongo_1...

> docker-agenda@0.0.1 start /app
> node app.js

Wed Jun 10 2015 13:44:55 GMT+0000 (UTC) 'say hello world in 5 seconds'
Wed Jun 10 2015 13:44:55 GMT+0000 (UTC) 'hello world'
```

MongoDBのコンテナに入りどんなレコードが作成されているか確認します。

``` bash
$ docker exec -it dockeragenda_mongo_1 bash
$ mongo agendadb
> show collections;
agendaJobs
system.indexes
> db.agendaJobs.find();
{ "_id" : ObjectId("55783f57c2f0820c0063b18d"), "name" : "hello world", "data" : { "time" : ISODate("2015-06-10T13:44:55.036Z") }, "type" : "normal", "priority" : 0, "nextRunAt" : null, "lastModifiedBy" : null, "lockedAt" : null, "lastRunAt" : ISODate("2015-06-10T13:45:00.041Z"), "lastFinishedAt" : ISODate("2015-06-10T13:45:00.046Z") }
```


### 繰り返す - every

[every(interval, name, [data])](https://github.com/rschmukler/agenda#everyinterval-name-data)メソッドは`name`のジョブを`interval`の間隔で繰り返し実行します。

```js
agenda.every('5 seconds', 'hello world', {time: new Date()});
agenda.start();
console.log(new Date(), 'say hello world every 5 seconds');
```

プログラムを実行するとすぐに1回目の`hello world`が表示され、以降は5秒間隔で出力されます。

``` bash
$ docker-compose run --rm agenda
Creating dockeragenda_mongo_1...

> docker-agenda@0.0.1 start /app
> node app.js

Wed Jun 10 2015 14:05:26 GMT+0000 (UTC) 'say hello world every 5 seconds'
Wed Jun 10 2015 14:05:26 GMT+0000 (UTC) 'hello world'
Wed Jun 10 2015 14:05:26 GMT+0000 (UTC) 'hello world'
```

MongoDBコンテナを確認してすると、ジョブの実行後にnextRunAtフィールドの値が5秒インクリメントされています。

``` bash
> db.agendaJobs.find();
{ "_id" : ObjectId("55784426dcc6bd7270dc5d81"), "name" : "hello world", "type" : "single", "data" : { "time" : ISODate("2015-06-10T14:05:26.161Z") }, "priority" : 0, "repeatInterval" : "5 seconds", "lastModifiedBy" : null, "nextRunAt" : ISODate("2015-06-10T14:06:11.264Z"), "lockedAt" : null, "lastRunAt" : ISODate("2015-06-10T14:06:06.264Z"), "lastFinishedAt" : ISODate("2015-06-10T14:06:06.266Z") }
> db.agendaJobs.find();
{ "_id" : ObjectId("55784426dcc6bd7270dc5d81"), "name" : "hello world", "type" : "single", "data" : { "time" : ISODate("2015-06-10T14:05:26.161Z") }, "priority" : 0, "repeatInterval" : "5 seconds", "lastModifiedBy" : null, "nextRunAt" : ISODate("2015-06-10T14:06:16.264Z"), "lockedAt" : null, "lastRunAt" : ISODate("2015-06-10T14:06:11.264Z"), "lastFinishedAt" : ISODate("2015-06-10T14:06:11.266Z") }
```


## ジョブの検索 - jobs

[jobs(mongoskin query)](https://github.com/rschmukler/agenda#jobsmongoskin-query)を実行してデータベースからジョブを検索してコールバックで取得します。[mongoskin](https://github.com/kissjs/node-mongoskin)のクエリが使えます。


```js
agenda.jobs({name: 'printAnalyticsReport'}, function(err, jobs) {
  // Work with jobs (see below)
});
```

## ジョブの更新

ジョブインスタンスのメソッドを実行して変更した内容は[save(callback)](https://github.com/rschmukler/agenda#savecallback)を実行してデータベースのジョブに保存して更新する必要があります。

```bash
job.repeatEvery('10 minutes');
job.save();
```

## ジョブの削除

ジョブの削除は[remove(callback)](https://github.com/rschmukler/agenda#removecallback)を実行します。

```js
job.remove(function(err) {
    if(!err) console.log("Successfully removed job from collection");
})
```

### アトミックな削除 - cancel

[cancel(mongoskin query, cb)](https://github.com/rschmukler/agenda#cancelmongoskin-query-cb)はジョブの検索(jobs())と削除(remove())をアトミックに行います。

```js
agenda.cancel({name: 'printAnalyticsReport'}, function(err, numRemoved) {
});
```

### すべて削除 - purge

[purge(cb)](https://github.com/rschmukler/agenda#purgecb)

設定変更などの結果ジョブが不要になったときは、[purge(cb)](https://github.com/rschmukler/agenda#purgecb)を使いジョブをデータベースから削除します。

``` js
agenda.purge(function(err, numRemoved) {
});
```