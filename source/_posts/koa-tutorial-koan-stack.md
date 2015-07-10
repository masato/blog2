title: "Koa入門 - Part5: 禅で公案なKOAN Stack"
date: 2015-07-04 19:43:00
tags:
 - Nodejs
 - iojs
 - ES6
 - Koa
 - KOAN
 - DockerCompose
description: KoaをES6で書くサンプルを探していたら、KOANというBoilerplateを見つけました。Koa、AngularJS、Node.jsのacronymのようです。個人的にAngularJSは苦手なのでKoaとNodeだけでも良かったのにと思いました。スウェーデンのTeoman Soygul氏が開発しています。禅の公案から来ているみたいです。KOANとKATAとかは西洋から見るとマトリックスの世界でしょうか。
---

[Koa](http://koajs.com/)をES6で書くサンプルを探していたら、[KOAN](https://github.com/soygul/koan)というBoilerplateを見つけました。Koa、AngularJS、Node.jsのacronymのようです。個人的にAngularJSは苦手なのでKoaとNodeだけでも良かったのにと思いました。スウェーデンのTeoman Soygul氏が開発しています。禅の公案から来ているみたいです。KOANとKATAとかは西洋から見るとマトリックスの世界でしょうか。

<!-- more -->


## KOANの特徴

[リポジトリ](https://github.com/soygul/koan)のREADMEには、MongoDBとWebSocketを含んだJavaScriptのフルスタックであると書いてあります。AngularJSとBootstrapを使いサーバーサイドレンダリングはありません。[デモアプリ](https://koan.herokuapp.com/signin.html)にはFacabookとGoogleのOAuthの認証が実装されています。[JWT](http://jwt.io/)を使っているようなのでよく理解したいです。

## プロジェクト

KoaはES6のジェネレータを使っています。KOANのKoaでは他のES6の機能は使っていないようです。io.jsまたはNode.jsの0.12と`--harmony`フラグが必要です。io.jsの2.3のDockerイメージを使います。今回作業をしたリポジトリは[こちら](https://github.com/masato/koan)です。

### git clone

Blueprintなので基本的にはgit cloneしてからプロジェクトのビジネスロジックを実装していくことになります。

```bash
$ cd ~/node_apps
$ git clone --depth 1 https://github.com/soygul/koan.git
```

### package.json

package.jsonのscriptsディレクティブは以下のようになっています。`npm start`でgruntが起動します。bowerはコンテナのrootが実行するので`--allow-root`を追加しておきます。

```json ~/node_apps/koan/package.json
...
  "scripts": {
    "start": "grunt",
    "test": "node --debug --harmony node_modules/grunt-cli/bin/grunt test",
    "postinstall": "bower install --allow-root",
    "postupdate": "bower update --allow-root"
  },
...
```

### gruntfile.js

gruntfile.jsのデフォルトでは、`nodemon`タスクでapp.jsの実行と`watch`タスクでライブリロードが動作する開発環境のサーバーが起動します。とても良いBlueprintなので参考になります。

```js ~/node_apps/koan/gruntfile.js
    watch: {
      client: {
        files: ['client/**', '!client/bower_components/**'],
        options: {
          livereload: true
        }
      },
      server: {
        files: ['.nodemon'],
        options: {
          livereload: true
        }
      }
    },
    nodemon: {
      dev: {
        script: 'app.js',
        options: {
          nodeArgs: ['--debug', '--harmony'],
          ignore: ['node_modules/**', 'client/**'],
          callback: function (nodemon) {
            fs.writeFileSync('.nodemon', 'started');
            nodemon.on('log', function (event) {
              console.log(event.colour);
            });
            nodemon.on('restart', function () {
              setTimeout(function () {
                fs.writeFileSync('.nodemon', 'restarted');
              }, 250);
            });
          }
        }
      }
    },
```

### Dockerfile

cloneしたディレクトリでDockerイメージを作成します。コンテナにディレクトリごとアタッチします。

```bash ~/node_apps/koan/Dockerfile
FROM iojs:2.3
MAINTAINER Masato Shimizu <ma6ato@gmail.com>

RUN mkdir -p /app
WORKDIR /app

COPY package.json /app/
RUN mkdir -p /dist/node_modules && \
    ln -s /dist/node_modules /app/node_modules && \
    npm install

EXPOSE 3000
COPY . /app
CMD ["npm","start"]
```

MongoDB用のライブラリは[co-mongo](https://github.com/thomseddon/co-mongo)です。MongoDBの2.x系に依存しているため[オフィシャルイメージ](https://registry.hub.docker.com/u/library/mongo/)は2.6を指定します。


### docker-compose.yml

docker-compose.ymlではデフォルトのkoanサービスとnpm-run-script用のnpmサービスを用意しました。マップはデフォルトの3000をホストが他で使っているため3030にマップします。35729はGruntのlivereloadで使用します。

```yml ~/node_apps/koan/docker-compose.yml
koan: &defaults
  build: .
  volumes:
    - .:/app
    - /etc/localtime:/etc/localtime:ro
  links:
    - mongo
  ports:
    - 3030:3000
    - 35729:35729
npm:
  <<: *defaults
  entrypoint: ["npm"]
mongo:
  image: mongo:2.6
  volumes:
    - ./mongo:/data/db
    - /etc/localtime:/etc/localtime:ro
```

### server/config/config.js

Docker Composeがサービス間の構成管理してくれるため、links経由で名前解決が簡単になります。MongoDBのURLはデフォルトではlocalhostです。ホスト名を`mongo`に変更します。


```js ~/node_apps/koan/server/config/config.js
  development: {
    app: {
      port: 3000
    },
    mongo: {
      url: 'mongodb://mongo:27017/koan-dev'
    },
```

## ビルドとコンテナ起動

### Dockerイメージ

Docker Composeを使ってkoanサービス用のDockerイメージのビルドします。.dockerignoreを用意してmongのデータベースディレクトリを無視しておきます。

```bash
$ cd ~/node_apps/koan
$ echo mongo > .dockerignore
$ docker-compose build koan
```

Dockerfileの`npm install`の後に、`.bowerrc`を使ったpostinstallが実行されないようです。docker-compose.ymlに定義した`npm`サービスを使って`npm run postinstall`を実行します。


```bash
$ docker-compose run --rm npm run postinstall
npm info it worked if it ends with ok
npm info using npm@2.11.1
npm info using node@v2.3.0
npm info postinstall koan@1.0.0

> koan@1.0.0 postinstall /app
> bower install --allow-root

? May bower anonymously report usage statistics to improve the tool over time? (Y/n)Y
...
jquery#2.1.4 client/bower_components/jquery

bootstrap#3.3.4 client/bower_components/bootstrap
└── jquery#2.1.4

font-awesome#4.3.0 client/bower_components/font-awesome
npm info ok
```

`.bowerrc`に定義したディレクトリにbower_componentsが作成されてクライアントのライブラリがインストールされました。

```bash
$ cat .bowerrc
{
  "directory": "client/bower_components"
}
$ ls client/bower_components/
angular          angular-elastic      angular-mocks  bootstrap         font-awesome  lodash
angular-animate  angular-loading-bar  angular-route  bootstrap-social  jquery
```

### 起動

カレントディレクトリにコンテナ内の`/dist/node_modules`ディレクトリへのシムリンクを作成します。docker-compose.ymlにサービスは2つ定義しているため`koan`サービスを明示的にupします。

```bash
$ ln -s /dist/node_modules .
$ docker-compose up koan
Recreating koan_mongo_1...
Creating koan_koan_1...
Attaching to koan_koan_1
koan_1 | npm info it worked if it ends with ok
koan_1 | npm info using npm@2.11.1
koan_1 | npm info using node@v2.3.0
koan_1 | npm info prestart koan@1.0.0
koan_1 | npm info start koan@1.0.0
koan_1 |
koan_1 | > koan@1.0.0 start /app
koan_1 | > grunt
koan_1 |
koan_1 | Running "concurrent:tasks" (concurrent) task
koan_1 | Running "watch" task
koan_1 | Running "nodemon:dev" (nodemon) task
koan_1 | Waiting...
koan_1 | [nodemon] v1.3.7
koan_1 | [nodemon] to restart at any time, enter `rs`
koan_1 | [nodemon] watching: *.*
koan_1 | [nodemon] starting `node --debug --harmony app.js`
koan_1 | Debugger listening on port 5858
koan_1 | Failed to load c++ bson extension, using pure JS version
koan_1 | KOAN listening on port 3000
```

ブラウザからDockerホストに接続します。Herokuにデプロイされている[デモアプリ]((https://koan.herokuapp.com/signin.html))と同じ画面が開きました。

![koan.png](/2015/07/04/koa-tutorial-koan-stack/koan.png)