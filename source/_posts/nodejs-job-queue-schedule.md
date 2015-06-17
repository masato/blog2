title: "Node.jsで使うJobQueueとcron型Scheduleのリソース"
date: 2015-06-05 21:13:01
tags:
 - Nodejs
 - JobQueue
 - cron
 - Schedule
description: Railsの本番環境でここ数年止まることなく安定してResqueとRakeのワーカーが動いているので満足しています。今はNode.jsで書きたいのでいわゆるJobQueueとかcronタイプのschedulerとか調べています。バックエンドにRedisは便利なのですがMongoDBの方が良いのかなと思います。MongoDBも小さく使う分には雑に扱っても安定している印象です。
---

Railsの本番環境でここ数年止まることなく安定してResqueとRakeのワーカーが動いているので満足しています。今はNode.jsで書きたいのでいわゆるJobQueueとかcronタイプのschedulerとか調べています。バックエンドにRedisは便利なのですがMongoDBの方が良いのかなと思います。MongoDBも小さく使う分には雑に扱っても安定している印象です。

<!-- more ->

## JobQueue

今ひとつJobQueueとcron型Schedulerの使い分けがハッキリしないのですが、Resqueの場合はバックグランドにまわしたいQueueと[resque-scheduler](https://github.com/resque/resque-scheduler)を組み合わせて要求は満たせていました。cronでスケジュールしたいけど、REST APIでアドホックにenqueueしたい。scheduleも途中でしてキャンセルしてrescheduleしたい。使うツールに悩みます。

### MongoDB

[agenda](https://github.com/rschmukler/agenda)は使っている人も多いみたいで、JobQueueとScheduleの両方に対応して今のところ一番良いのではないかと思います。[Jobukyu](https://github.com/rschmukler/agenda)はREST APIで操作できるので要件的にこっちかも知れません。

* [agenda](https://github.com/rschmukler/agenda)
* [monq](https://github.com/scttnlsn/monq)
* [mongodb-queue](https://github.com/chilts/mongodb-queue)
* [Jobukyu](https://github.com/rschmukler/agenda)

### Redis

経験則だとResque型は最悪redis-cliでどうにでも操作できるので安心できます。[Kue](https://github.com/Automattic/kue/)はREST APIも使え、Expressや[Sails.js](http://sailsjs.org/)とのインテグレーションの実績もあるので本格的なWebアプリをNode.jsで書くときに使いたいと思います。

* [Kue](http://automattic.github.io/kue/)
* [Bull](https://github.com/OptimalBits/bull)
* [Barbeque](https://github.com/pilwon/barbeque)
* [node-resque](https://github.com/taskrabbit/node-resque)
* [Coffee-Resque](https://github.com/technoweenie/coffee-resque)

## cron型のschedule

cronタイプのschedulerだと[node-schedule](https://github.com/node-schedule/node-schedule)が使いやすそうです。ドキュメントが少ないのが残念ですがテストケースを読むとジョブのキャンセルとかできるみたいです。

* [node-cron](https://github.com/ncb000gt/node-cron)
* [node-schedule](https://github.com/node-schedule/node-schedule)
* [Later.js](https://github.com/bunkat/later)



