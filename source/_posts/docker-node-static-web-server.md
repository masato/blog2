title: "Dockerでnode-stacicの静的コンテンツ用Webサーバーを起動する"
date: 2015-04-21 21:00:35
tags:
 - Docker
 - Nodejs
 - node-static
 - JavaScript
description: テスト用に手軽な静的Webサーバーを使いたかったのでnode-staticをDockerコンテナで起動しました。ちょっとしたJavaScriptのアプリを簡単にデプロイして確認することができます。
---

テスト用に手軽な静的Webサーバーを使いたかったので[node-static](https://github.com/cloudhead/node-static)をDockerコンテナで起動しました。ちょっとしたJavaScriptのアプリを簡単にデプロイして確認することができます。

<!-- more -->

## node-staticコンテナ

最初にプロジェクトに作成したファイル一覧です。

``` bash
$ cd ~/node_apps/node-static
$ tree
.
├── Dockerfile
├── app.js
├── package.json
└── public
    └── index.html
```

Dockerfileは[google/nodejs-runtime](https://registry.hub.docker.com/u/google/nodejs-runtime/)をベースイメージに使います。

``` bash ~/node_apps/node-static/Dockerfile
FROM google/nodejs-runtime
VOLUME /app/public
EXPOSE 80
```

app.jsのメインプログラムでは静的ファイルをデプロイするディレクトリとして`public`を指定します。

``` js ~/node_apps/node-static/app.js
var static = require('node-static');
var file = new static.Server('./public');

require('http').createServer(function (request, response) {
    request.addListener('end', function () {
        file.serve(request, response);
    }).resume();
}).listen(80);

console.log("Server running at http://localhost");
```

`package.json`では`node-static`パッケージをインストールします。

``` json ~/node_apps/node-static/package.json
{
  "name": "node-static-app",
  "description": "node static app",
  "version": "0.0.1",
  "private": true,
  "dependencies": {
    "node-static": "0.7.6"
  },
  "scripts": {"start": "node app.js"}
}
```

[A Minimal HTML Document (HTML5 Edition)](http://www.sitepoint.com/a-minimal-html-document-html5-edition/)を参考にしてミニマルな`index.html`を用意します。

``` html ~/node_apps/node-static/public/index.html
<!DOCTYPE html>
<html lang="ja">
  <head>
    <meta charset="utf-8">
    <title>title</title>
  </head>
  <body>
    <p>hello world</p>
  </body>
</html>
```

## コンテナの起動と確認

Dockerイメージをビルドします。

``` bash
$ docker build -t node-static . 
```

Dockerコンテナを起動します。

``` bash
$ docker run -d \
  --name node-static \
  -p 80:80 \
  -v $PWD/public:/app/public \
  node-static
```

localhostから簡単に確認してみます。

```
$ curl localhost
<!DOCTYPE html>
<html lang="ja">
  <head>
    <meta charset="utf-8">
    <title>title</title>
  </head>
  <body>
    <p>hello world</p>
  </body>
</html>
```

`public`ディレクトリはVOLUMEにマウントしているのでDockerコンテナが起動中にファイルを編集することができます。
