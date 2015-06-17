title: "Node.js LoopBack on Docker - Part1: MongoDBのコンテナを作成"
date: 2014-12-24 22:18:41
tags:
 - MBaaS
 - LoopBack
 - StrongLoop
 - Nodejs
 - Isomorphic
 - MongoDB
 - DockerDevEnv
 - MEAN
description: LoopBackはNode.jsのオープンソースのMBaaSです。開発元のStrongLoopはエンタープライズ向けにNode.jsのAPIプラットフォームを提供しています。この前MBaaSのサービスを調べたところ、オープンソースのMBaaSもいくつかありました。LoopBackはMBaaSとしても一般的な機能をそなえていますが、ExpressでできたSwagger 2.0準拠のAPI開発用のNode.jsフレームワークとなっています。モデルを定義するとREST APIを自動生成してくれるのが特徴です。クライアントSDKはAndroid, iOS, AngularJSがあります。特にAngularJS SDKはIsomorphicなのでサーバーとクライアントで同じモデルを共有できます。
---

LoopBackはNode.jsのオープンソースのMBaaSです。開発元のStrongLoopはエンタープライズ向けにNode.jsのAPIプラットフォームを提供しています。この前[MBaaSのサービス](/2014/12/22/mbaas-html5-hybrid-mobile-apps-references/)を調べたところ、オープンソースのMBaaSもいくつかありました。LoopBackはMBaaSとしても一般的な機能をそなえていますが、[Express](http://expressjs.com/)でできた[Swagger 2.0](http://swagger.io/index.html)準拠のAPI開発用Node.jsフレームワークとなっています。モデルを定義するとREST APIを自動生成してくれるのが特徴です。クライアントSDKはAndroid, iOS, AngularJSがあります。特にAngularJS SDKはIsomorphicなのでサーバーとクライアントで同じモデルを共有できます。

<!-- more -->

### MongoDBコンテナの作成

LoopBackのバックエンドのデータストアとしてMongoDBを使うことにします。まずはLoopBackコンテナの作成の前に準備します。Docker Hub Registryで検索するとTrusted の[dockerfile/mongodb](https://registry.hub.docker.com/u/dockerfile/mongodb/)が良さそうです。

Dockerホストにボリュームのディレクトリを作成して、とりあえずSmallサイズのDockerコンテナを起動します。

``` bash
$ sudo mkdir -p /opt/mongo/db
$ docker run -d -p 27017:27017 \
  -v  /opt/mongo/db:/data/db \
  --name mongodb \
  dockerfile/mongodb \
  mongod --smallfiles
```

### mongoコマンド用コンテナ

mongoコマンドを実行するため、Dockerコンテナを起動します。MongoDBコンテナにlinkして動作確認をします。
Dockerを使うと開発用のMongoDBサーバーが簡単に用意できました。とてもお手軽です。

``` bash
$ docker run -it --rm \
  --link mongodb:mongodb \
  dockerfile/mongodb \
  bash -c 'mongo --host mongodb'
MongoDB shell version: 2.6.6
connecting to: mongodb:27017/test
Welcome to the MongoDB shell.
For interactive help, type "help".
For more comprehensive documentation, see
        http://docs.mongodb.org/
Questions? Try the support group
        http://groups.google.com/group/mongodb-user
> show databases;
admin  (empty)
local  0.031GB
```
