title: "Node-RED on Docker - Part1: インストール"
date: 2015-02-07 03:23:51
tags:
 - Docker
 - Node-RED
 - Nodejs
description: Node.jsでつくるIoTのサーバーサイドにNode-REDを使います。MQTTからメッセージを取得してMongoDBに保存する処理をワイヤリングします。IoTプラットフォームはDockerコンテナで構築していきます。
---

Node.jsでつくるIoTのサーバーサイドにNode-REDを使います。MQTTからメッセージを取得してMongoDBに保存する処理をワイヤリングします。IoTプラットフォームはDockerコンテナで構築していきます。

<!-- more -->

## google/nodejs-runtimeのDockerイメージ

[google/nodejs-runtime](https://registry.hub.docker.com/u/google/nodejs-runtime/)を使う場合の前提条件は以下です。

* package.jsonがあること
* エントリーポイントのserver.jsまたは、package.jsonにscriptsの記述があること
* アプリのportは8080でlistenしていること

## Node-REDのGitHubリポジトリ

Node-REDの[リポジトリ](https://github.com/node-red/node-red)を確認して、google/nodejs-runtimeの仕様にあわせます。

[package.json](https://github.com/node-red/node-red/blob/master/package.json)にエントリーポイントの指定があります。

``` js package.json
...
    "scripts"      : {
        "start": "node red.js",
        "test": "./node_modules/.bin/grunt"
    },
...
```

エントリーポイントの[red.js](https://github.com/node-red/node-red/blob/master/red.js)を見ると、設定ファイルはデフォルトでカレントディレクトリのsettings.jsを参照しています。

``` js red.js
...
var settingsFile = "./settings";
...
Usage: node red.js [-v] [-?] [--settings settings.js] [flows.json]
...
```

[settings.js](https://github.com/node-red/node-red/blob/master/settings.js)のポート指定は1880なのでEXPOSEに追加します。

``` js settings.js
...
module.exports = {
    // the tcp port that the Node-RED web server is listening on
    uiPort: 1880,
...
```

## Dockerfileのビルド

Node-REDを`git clone`します。

``` bash
$ cd ~/docker_apps
$ git clone https://github.com/node-red/node-red.git
$ cd node-red
```

Dockerfileを作成して、`docker build`を実行します。

``` bash
$ cat << 'EOF' > Dockerfile
FROM google/nodejs-runtime
EXPOSE 1880
EOF
$ docker build -t node-red .
```

## コンテナの起動

使い捨てのコンテナを起動して、1880ポートをDockerホストにマップします。

``` bash
$ docker run --rm --name node-red -p 1880:1880 node-red
```

ブラウザで動作確認をします。

http://10.1.3.67:1880

![node-red.png](/2015/02/07/node-red-on-docker-install/node-red.png)
