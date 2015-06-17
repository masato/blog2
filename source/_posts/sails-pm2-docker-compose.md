title: "Sails.jsの開発をDocker Composeで行いPM2でファイル監視と再起動をする"
date: 2015-04-30 11:04:32
tags:
 - PM2
 - Sails
 - Docker
 - DockerCompose
 - SaneStack
description: Sane Stackの動作を理解するために、前回はEmber CLIだけのDocker Composeを作成しました。今度はSails.jsだけ使ったDocker Composeを用意して簡単なアプリを作成してみます。
---

[Sane Stack](http://sanestack.com/)の[動作](/2015/04/24/sane-stack-emberjs-salisjs-docker-compose-quickstart/)を理解するために、[前回](/2015/04/25/docker-compose-ember-cli-init/)はEmber CLIだけのDocker Composeを作成しました。今度はSails.jsだけ使ったDocker Composeを用意して簡単なアプリを作成してみます。

<!-- more -->

## プロジェクト

最初に適当なディレクトリを作成します。`server`ディレクトリは`sails new`コマンドを使い、空のカレントディレクトリにプロジェクトを作成するために用意しておきます。

``` bash
$ cd ~/node_apps/sails-spike
$ mkdir -p server pm2/log
```

最終的に次のようなディレクトリ構成になります。

``` bash
$ cd ~/node_apps/sails-spike
$ tree -L 2
.
├── Dockerfile
├── docker-compose.yml
├── package.json
├── pm2
│   ├── config
│   └── log
└── server
    ├── Gruntfile.js
    ├── README.md
    ├── api
    ├── app.js
    ├── assets
    ├── config
    ├── node_modules
    ├── package.json
    ├── tasks
    └── views
```

### DockerとDocker Compose

Dockerfileは[0.12/onbuild](https://registry.hub.docker.com/_/node/)をベースイメージに使います。コンテナ内でsailsコマンドを実行するため、作業ユーザーと同じUIDでユーザーを作成します。マウントしたDockerホストのディレクトリ内でファイルを編集できるようにします。

ENTRYPOINTにsailsコマンドを指定して、CMDはhelpにしています。コンテナをrunしたとき引数を渡さないと`sails help`が実行されます。`docker-compose`コマンドの引数には`sails`サービスに続けて`sails`サブコマンドを指定します。

``` bash ~/node_apps/sails-spike/Dockerfile
FROM node:0.12-onbuild
MAINTAINER Masato Shimizu <ma6ato@gmail.com>

RUN npm install -g sails grunt bower pm2 npm-check-updates

RUN adduser --disabled-password --gecos '' --uid 1000 docker && \
  adduser docker sudo && \
  echo '%sudo ALL=(ALL) NOPASSWD:ALL' >> /etc/sudoers && \
  mkdir -p /app && \
  chown docker:docker /app

USER docker
WORKDIR /app

EXPOSE 1337
ENTRYPOINT ["/usr/local/bin/sails"]
CMD ["help"]
```

docker-compose.ymlはYAMLのマージを利用しています。`&defaults`にしている`sails`サービスのイメージは事前にローカルにビルドしておきます。

```bash
$ cd ~/node_apps/sails-spike/
$ docker build -t sails .
```

`server`サービスではPM2を使ってSailsアプリを起動します。Dockerなので`--no-daemon`フラグを付けてフォアグラウンドで実行します。

```yaml  ~/node_apps/sails-spike/docker-compose.yml
sails: &defaults
  image: sails
  volumes:
    - $PWD/server:/app
    - $PWD/pm2:/opt/pm2

server:
  <<: *defaults
  entrypoint: ["pm2","--no-daemon","start","/opt/pm2/config/development.json"]
  ports:
    - 1337:1337
    
npm:
  <<: *defaults
  entrypoint: ['/usr/local/bin/npm']

bower:
  <<: *defaults
  entrypoint: ['/usr/local/bin/bower', '--allow-root']
```

ベースイメージのONBUILDで実行される`package.json`です。sailsをインストールします。

```json  ~/node_apps/sails-spike/package.json
{
  "name": "sails-spike",
  "description": "sails spike project.",
  "version": "0.0.1",
  "dependencies": {
    "sails": "0.11.0"
  },
  "scripts": {"start": "node app.js"}
}
```

### PM2

Sailsプロセスの管理は[PM2](https://github.com/Unitech/pm2)を使います。[JSON](https://github.com/Unitech/PM2/blob/master/ADVANCED_README.md#json-app-declaration)の設定ファイルを用意します。`watch`に監視したいディレクトリを指定するとファイルの更新時に自動的にアプリを再起動してくれます。

``` json ~/node_apps/sails-spike/pm2/config/development.json
{
  "apps": [{
    "name"      : "sails-app",
    "cwd"       : "/app",
    "script"    : "app.js",
    "watch"     : ["api", "config"],
    "out_file"  : "/opt/pm2/log/sails-app.stdout.log",
    "error_file": "/opt/pm2/log/sails-app.stderr.log",
    "env": {
      "NODE_ENV": "development",
      "PORT"    : 1337
    }
  }]
}
```


## sailsコマンド

sailsコマンドをDockerコンテナ内で実行します。最初にSailsのバージョンを確認します。`--rm`フラグを付けてコマンド実行後にコンテナが破棄されるようにします。

``` bash
$ docker-compose run --rm sails --version
0.11.0
Removing sailsspike_sails_run_1...
```

同様にdocker-composeに定義した`npm`サービスを使ってnpmコマンドのバージョンを確認します。

``` bash
$ docker-compose run --rm npm --version
2.8.4
Removing sailsspike_npm_run_1...
```

ちょっとわかりづらいですが、`--entrypoint`フラグを上書きして任意のコマンドも実行できます。結果的にはYAMLでマージした`npm`サービスと同じ動作になります。

``` bash
$ docker-compose run --rm --entrypoint="/usr/local/bin/node" sails --version
v0.12.2
       Removing sailsspike_sails_run_4...
```       
       
`server`ディレクトリはコンテナの`/app`ディレクトリにマウントしています。ここに`sails new`コマンドでアプリを作成します。

``` bash
$ docker-compose run --rm sails new
info: Created a new Sails app `app`!
Removing sailsspike_sails_run_1...
```

`server`サービスをupします。

``` bash
$ docker-compose up server 
```

ブラウザから確認します。

http://10.3.0.165:1337/

![sails-home.png](/2015/04/30/sails-pm2-docker-compose/sails-home.png)




## APIの作成

`sails generate api`コマンドからコントローラーとモデルを生成します。

``` bash
$ docker-compose run --rm sails generate api user
info: Created a new api!
Removing sailsspike_sails_run_1...
```

`genarate api`ではモデルのattributesを渡せません。生成されたUser.jsモデルを編集してattributesを追加します。

``` js ~/node_apps/sails-spike/server/api/models/User.js
module.exports = {
  attributes: {
    name: 'string',
    age: 'integer'
  }
};
```

初回起動時に`Excuse my interruption`とデータベースのマイグレーションをどうするかダイアログが表示されます。`sails lift`している場合は`config/models.js`の設定が入りますが、foreverやpm2で起動していると編集されません。

[Q&A: Sails.js won’t run through forever.js](http://blog.spoonx.nl/qa-sails-js-wont-run-through-forever-js/)

手動でアンコメントします。

``` js ~/node_apps/sails-spike/server/config/models.js
module.exports.models = {
  connection: 'localDiskDb',
  migrate: 'alter'
};
```

pm2がファイルの編集を検知してアプリを再起動してくれます。

``` bash
Change detected for app name: sails-app - restarting
```

### REST API

curlコマンドからJSON形式でPOSTしてレコードを作成します。HTTPヘッダにcontent-typeを指定します。

``` bash
$ curl -X POST \
  http://10.3.0.165:1337/user \
  -H "Accept: application/json" \
  -H "Content-Type: application/json" \
  -d '{"name":"pochi","age": 3 }'
{
  "name": "pochi",
  "age": 3,
  "createdAt": "2015-04-30T12:30:03.135Z",
  "updatedAt": "2015-04-30T12:30:03.135Z",
  "id": 1
}
```

同様にcurlから登録したレコードをGETします。

``` bash
$ curl -X GET \
  http://10.3.0.165:1337/user/1
{
  "name": "pochi",
  "age": 3,
  "createdAt": "2015-04-30T12:30:03.135Z",
  "updatedAt": "2015-04-30T12:30:03.135Z",
  "id": 1
}
```


