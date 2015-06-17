title: "RESTクライアントのCLIをNode.jsのCommander.jsを使って作成する"
date: 2015-05-15 15:59:55
tags:
 - DockerCompose
 - Nodejs
 - CLI
 - Mesublu
 - Commanderjs
description: 開発中のRESTクライアントはcurlコマンドを使っていましたが、他の人に使ってもらうためにコマンドラインツールが必要になります。サーバーと同じ開発言語のNode.jsで実装します。サーバーやデータベースのコンテナとセットにしてDocker Composeで配布する予定ですが、動作確認のためとりあえず単体で動くサンプルを作成しました。
---

開発中のRESTクライアントはcurlコマンドを使っていましたが、他の人に使ってもらうためにコマンドラインツールが必要になります。サーバーと同じ開発言語のNode.jsで実装します。サーバーやデータベースのコンテナとセットにしてDocker Composeで配布する予定ですが、動作確認のためとりあえず単体で動くサンプルを作成しました。

<!-- more -->

## プロジェクトの作成

プロジェクトのディレクトリを作成して以下のようなファイルを作成します。

``` bash
$ cd ~/node_apps/iot-util
$ tree .
.
├── Dockerfile
├── app.js
├── commands
│   └── status.js
├── docker-compose.yml
├── node_modules -> /dist/node_modules
└── package.json
```

Dockerホストのプロジェクトのディレクトリに`/dist/node_modules`へのシムリンクを作成します。

```bash
$ cd ~/node_apps/iot-util
$ ln -s /dist/node_modules .
```

Node.jsの実行はDockerコンテナで行いますが、イメージ再作成とnode_modulesの再ビルドを回避するためnode_modulesはDockerホストのボリュームにマウントして使います。ベースイメージはオフィシャルの[node:0.12](https://github.com/joyent/docker-node/blob/530696ff3003a169521a61a0e04bbf980b7228d5/0.12/Dockerfile)を使います。

```bash  ~/node_apps/iot-util/Dockerfile
FROM node:0.12
MAINTAINER Masato Shimizu <ma6ato@gmail.com>

RUN mkdir -p /app
WORKDIR /app

COPY package.json /app/
RUN mkdir -p /dist/node_modules && ln -s /dist/node_modules /app/node_modules && \
  npm install
COPY . /app
ENTRYPOINT ["npm", "start"]
```

Docker Composeを使ってビルドとCLIの実行をします。環境変数の`MESHBLU_URL`は後でdocker-compose.ymlのlinksから取得します。

```yml ~/node_apps/iot-util/docker-compose.yml
iotutil:
  build: .
  volumes:
    - .:/app
  environment:
    - MESHBLU_URL=https://10.3.0.165/
```

## Commander.jsを使う

Node.jsにはCLIを作るためのパッケージがいくつかありますが、よく使われている[Commander.js](https://github.com/tj/commander.js)で実装していきます。

```json ~/node_apps/iot-util/package.json
{
  "name": "iot-util",
  "description": "IoT Toolkit",
  "version": "0.0.1",
  "private": true,
  "dependencies": {
    "commander": "^2.8.1",
    "request": "^2.55.0"
  },
  "scripts": {"start": "node app.js"}
}
```

以下のサイトを参考にしてエントリーポイントのapp.jsに必要なコードを書きます。

* [Conquering Commander.js](http://slides.com/timsanteford/conquering-commander-js#/)
* [Command-line utilities with Node.js](http://cruft.io/posts/node-command-line-utilities/)
* [Writing a Command Line Node Tool](http://javascriptplayground.com/blog/2015/03/node-command-line-tool/)


```js ~/node_apps/iot-util/app.js
#!/usr/bin/env node

var program = require('commander')
  , status = require('./commands/status.js');

program
    .version('0.0.1')
    .usage('<command>');

program
    .command('status')
    .description('show status')
    .action(status.commandStatus);

program.parse(process.argv);
if (process.argv.length < 2) {
    console.log('You must specify a command'.red);
    program.help();
}

exports.program = program;
```

個別のコマンドはファイル名にコマンド名をつけて、`commands`ディレクトリに配置します。今回はサンプルにstatus.jsを作成しました。サーバーは自己署名の証明書を使っています。Node.jsの[request](https://github.com/request/request)パッケージの場合は`curl --insecure`に相当する`rejectUnauthorized: false`のオプションを付けます。

```js ~/node_apps/iot-util/commands/status.js
var request = require('request')
  , path = require('path');
  
module.exports = {
    commandStatus: function(symbol, command) {

        var filename = path.basename(__filename),
            cmd = filename.substr(0, filename.lastIndexOf('.'));

        var options = {
            url: process.env.MESHBLU_URL + cmd ,
            agentOptions: {
                rejectUnauthorized: false
            }
        }

        request.get(options,function(error, response, body) {
            if (!error && response.statusCode == 200){
                var body = JSON.parse(body);
                console.log(body);
            } else if (error) {
                console.log('Error: ' + error);
            }
        });
    }
}
```


## CLIの実行

Dockerイメージのビルドをします。

``` bash
$ cd ~/node_apps/iot-util
$ docker-compose build
```

コマンドファイルと同じ名前の引数を渡してMeshbluのstatusを取得してみます。

``` bash
$ docker-compose run --rm iotutil status

> iot-util@0.0.1 start /app
> node app.js "status"

{ meshblu: 'online' }
Removing iotutil_iotutil_run_1...
```

ヘルプを実行する場合は、`--`に続けて`--help`フラグを標準入力から取得します。

``` bash
$ docker-compose run --rm iotutil -- --help

> iot-util@0.0.1 start /app
> node app.js "--help"


  Usage: app <command>


  Commands:

    status   show status

  Options:

    -h, --help     output usage information
    -V, --version  output the version number

Removing iotutil_iotutil_run_1...
```