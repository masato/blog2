title: "Sane Stackを使いEmber.jsとSails.jsアプリをDocker Compose上で開発する"
date: 2015-04-24 19:09:20
categories:
 - Docker
tags:
 - Docker
 - DockerCompose
 - Sailsjs
 - Emberjs
 - ember-cli
 - SaneStack
 - Nodejs
 - CLI
description: Sane StackはJavaScriptフルスタックのフレームワークとCLIです。MEANが有名ですが、こちらはSails.jsとEmber.jsを採用しています。また開発にDocker Composeを取り入れたワークフローも使えるのが特徴的です。ただしDocker Composeの権限周りの条件がなかなか厳しいです。修正が追いついていないライブラリもあってまだ安定していないようです。
---

[Sane Stack](http://sanestack.com/)はJavaScriptフルスタックのフレームワークとCLIです。[MEAN](http://mean.io/)が有名ですが、こちらはSails.jsとEmber.jsを採用しています。また開発にDocker Composeを取り入れたワークフローも使えるのが特徴的です。ただしDocker Composeの権限周りの条件がなかなか厳しいです。修正が追いついていないライブラリもあってまだ安定していないようです。

<!-- more -->

## インストール

### DockerとDocker Compose

DockerとDocker Composeをインストールします。rootユーザーで実行します。ホストマシンは`Ubuntu 14.04.1`を用意しました。

ワンライナーインストーラーでは、すでにDockerがインストールされている場合はアップデートを中止することもできます。

``` bash
$ wget -qO- https://get.docker.com/ | sh
Warning: "docker" or "lxc-docker" command appears to already exist.
Please ensure that you do not already have docker installed.
You may press Ctrl+C now to abort this process and rectify this situation.
+ sleep 20
```

DockerComposeをインストールします。

``` bash
$ curl -L https://github.com/docker/compose/releases/download/1.2.0/docker-compose-`uname -s`-`uname -m` > /usr/local/bin/docker-compose; chmod +x /usr/local/bin/docker-compose
```

バージョンを確認します。

``` bash
$ docker version
Client version: 1.6.0
$  docker-compose --version
docker-compose 1.2.0
```

### Node.js

Docker Composeの起動とnpmのインストールの権限まわりに問題があったので現状ではroot権限でSane Stackは操作する必要があります。またember-cliは将来的にapt-getでインストールされる`v0.10.25`をサポートしないという警告がでるので、[NodeSource](https://nodesource.com/)から`v0.12`をインストールすることにします。

[Node.js v0.12, io.js, and the NodeSource Linux Repositories](https://nodesource.com/blog/nodejs-v012-iojs-and-the-nodesource-linux-repositories)にインストール方法が書いてあります。

``` bash
$ curl -sL https://deb.nodesource.com/setup_0.12 | sudo bash -
$ sudo apt-get install -y nodejs
$ node -v
v0.12.2
$ npm -v
2.8.4
```

### Sane Stack

Sane Stackのインストールはなぜかnpmの順番が重要です。同時に列挙してインストールすると失敗してしまうので以下の順番でインストールします。

``` bash
$ sudo npm install -g sails 
$ sudo npm install -g ember-cli 
$ sudo npm install -g sane-cli 
```


## クイックスタート

[QUICKSTART](http://sanestack.com/#sane-stack-quickstart)を読みながらプロジェクトを作っていきます。`--docker`フラグをつけてアプリを作成します。Docker Composeを使って起動するので今のところroot権限が必要になります。

``` bash
$ cd ~/node_apps
$ sudo sane new sane-spike --docker
...
Installed packages for tooling via npm.
Installed browser packages via Bower.
Sane Project 'sane-spike' successfully created.
```

[error: Error: Cannot find module 'lodash/Lang/isObject' #141](https://github.com/artificialio/sane/issues/141)にissueがあるように、sails-hook-devが古くエラーが発生します。最新版がnpmからインストールできないためソースからインストールします。

``` bash
$ cd ~/node_apps/sane-spike/server
$ sudo npm i balderdashy/sails-hook-dev --save 
sudo npm i balderdashy/sails-hook-dev --save
sails-hook-dev@1.0.0 node_modules/sails-hook-dev
├── pretty-bytes@1.0.4 (get-stdin@4.0.1, meow@3.1.0)
└── fs-extra@0.16.5 (jsonfile@2.0.0, graceful-fs@3.0.6, rimraf@2.3.2)
```


アプリのディレクトリに移動してモデルとコントローラーを作成します。

``` bash
$ cd cd ~/node_apps/sane-spike
$ sudo sane generate resource user name:string age:number
info: Created a new model ("User") at api/models/User.js!
info: Created a new controller ("user") at api/controllers/UserController.js!
version: 0.2.3
Could not find watchman, falling back to NodeWatcher for file system events.
Visit http://www.ember-cli.com/#watchman for more info.
installing
  create app/models/user.js
installing
  create tests/unit/models/user-test.js
installing
  create app/routes/user.js
  create app/templates/user.hbs
installing
  create tests/unit/routes/user-test.js
```

`sane up`コマンドを実行してDocker Composeからアプリを起動します。Sails.jsサーバーは1337ポート、Ember.jsの開発サーバーは4200ポートで起動します。

``` bash
$ sudo sane up
docker-compose
Creating sanespike_server_1...
Attaching to sanespike_server_1
...
```

### APIを使う

curlコマンドを使ってJSONデータをPOSTします。
 
``` basg
$ curl -X POST \
  http://localhost:1337/api/v1/users \
  -d '{"user":{"name":"pochi","age":"3"}}' \
  -H "Accept: application/json" \
  -H "Content-type: application/json"
{
  "user": {
    "name": "pochi",
    "age": 3,
    "createdAt": "2015-04-29T14:32:43.521Z",
    "updatedAt": "2015-04-29T14:32:43.521Z",
    "id": 1
  }
}
```

同様にcurlコマンドでPOSTしたデータを取得することができます。

``` bash
$ curl -X GET http://localhost:4200/api/v1/users
{
  "users": [
    {
      "name": "pochi",
      "age": 3,
      "createdAt": "2015-04-29T14:32:43.521Z",
      "updatedAt": "2015-04-29T14:32:43.521Z",
      "id": 1
    }
  ]
}
```



