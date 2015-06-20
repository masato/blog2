title: "Hapi.js with Socket.IO - Part1: Hello World!"
date: 2015-06-14 20:47:06
tags:
 - Hapijs
 - SocketIO
 - dotenv
description: Hapi.jsのREST APIが一通り動くようになったので次はHapi.jsにSocket.IOサーバーを作成してみます。Using hapi.js with Socket.ioにとても良いSocket.IOの解説があります。著者のMatt Harrison氏はManningでHapi.js in ActionをMEAPで執筆中です。ブログが非常にわかりやすいので著書も期待できます。
---

[Hapi.js](http://hapijs.com/)のREST APIが[一通り動く](/2015/06/13/hapijs-plugins/)ようになったので次はHapi.jsにSocket.IOサーバーを作成してみます。[Using hapi.js with Socket.io](http://matt-harrison.com/using-hapi-js-with-socket-io/)にとても良いSocket.IOの解説があります。著者のMatt Harrison氏はManningで[Hapi.js in Action](http://www.manning.com/harrison/)をMEAPで執筆中です。ブログが非常にわかりやすいので著書も期待できます。

<!-- more -->

## リソース

Hapi.jsとSocket.IOの使い方は以下のサイトを参考にして勉強します。

* [Using hapi.js with Socket.io](http://matt-harrison.com/using-hapi-js-with-socket-io/)
* [SOCKET.IO WITH HAPI.JS](http://www.appsaloon.be/blog/socket-io-with-hapi-js/)
* [Hapi.js with Socket.io — Where is socket.io.js?](http://stackoverflow.com/questions/18343509/hapi-js-with-socket-io-where-is-socket-io-js)
* [Hapi + socket.io Example](https://github.com/expr/hapi-socketio-example)
* [Understanding Socket.IO](https://nodesource.com/blog/understanding-socketio)


## プロジェクト

適当なディレクトリにプロジェクトを作成します。リポジトリは[こちら](https://github.com/masato/docker-hapi-socketio)です。

```bash
$ cd ~/node_apps/docker-hapi-socketio
$ tree -a -L 2
.
├── .dockerignore
├── .env
├── .gitignore
├── Dockerfile
├── README.md
├── app.js
├── docker-compose.yml
├── node_modules -> /dist/node_modules
├── npm-debug.log
├── package.json
└── templates
    └── index.html
```

以下のバージョンのパッケージを使います。

```json ~/node_apps/docker-hapi-socketio/package.json
{
  "name": "docker-hapi-socketio",
  "description": "docker-hapi-socketio",
  "version": "0.0.1",
  "private": true,
  "dependencies": {
      "hapi": "^8.6.1",
      "socket.io": "^1.3.5",
      "handlebars": "^3.0.3",
      "dotenv": "^1.1.0"
  },
  "scripts": {"start": "node app.js"}
}
```


## サーバーサイド

### hapi serverのlisnterはhttp server

hapi server(`var server = new Hapi.Server();`)にconnectionを作成(`server.connection({ port: 4000 });`)すると、内部で新しいNode.jsのhttp server(`var listener = require('http').createServer(handler);`)が作成されます。このhttp serverインスタンスはlistenerプロパティ(`var listener = server.listener;`)になります。

### Socket.IOに渡すappもhttp server

Socket.IOをrequireするときに渡す`var io = require('socket.io')(app);`のapp変数もhapi serverのlistenerと同様にhttp server(`var app = require('http').createServer(handler);`)です。ただし以下のようにhapi serverにconnectionを作成して作成した8080ポートの`hapi server.listener`(`node http server`)をSocket.IOに渡すとセットアップ処理で`node http server`のrequestイベントのリスナーがすべて削除されてしまいます。

```js
var Hapi = require('hapi')
var server = new Hapi.Server();
server.connection({ port: 8080 });

var io = require('socket.io')(server.listener);

io.on('connection', function (socket) {
  console.log('connected');
});

server.start();
```

### API用のhttp serverとSocket.IO用のhttp server

hapiはportを指定して内部で複数の`node http server`を起動することができます。下の例では1つのhapi serverでREST APIを8080ポートでLISTENするHTTPサーバーと、8080ポートでLISTENするSocket.IOサーバーの2つが起動しています。HTTPサーバーはいまのところ`/`からindex.htmlのエントリポイントを提供するだけでREST APIのRouteは実装していません。

```js ~/node_apps/docker-hapi-socketio/app.js
'user strict';

var Hapi = require('hapi'),
    server = new Hapi.Server(),
    Path = require('path');
require('dotenv').load();

server.connection({ port: process.env.API_PORT, labels: ['api'] });
server.connection({ port: process.env.SOCKETIO_PORT, labels: ['twitter'] });

server.views({
    engines: {
        html: require('handlebars')
    },
    path: Path.join(__dirname, 'templates')
});

server.select('api').route({
    method: 'GET',
    path: '/',
    handler: function (request, reply) {
        reply.view('index',
                   { socketio_host: (process.env.PUBLIC_IP+':'
                                     +process.env.SOCKETIO_PORT)});
    }
});

var io = require('socket.io')(server.select('twitter').listener);
io.on('connection', function (socket) {
    console.log('connected!');
    var tweet = {text: 'hello world!'};
    var interval = setInterval(function () {
        socket.emit('tweet', tweet);
    }, 5000);

    socket.on('disconnect', function () {
        clearInterval(interval);
    });
});

server.start();
```

## クライアントサイド

クライアントサイドはSPAで実装します。HTTPサーバー(8000)がserveするindex.htmlがエントリポイントになります。index.htmlからSocker.IOサーバー(8080)がserveする`socket.io.js`のクライアントライブラリをロードしてSocket.IOサーバー(8080)に接続します。socket変数の`connect`と`tweet`イベントにlistenerをアタッチします。

```html ~/node_apps/docker-hapi-socketio/templates/index.html
<!DOCTYPE html>
<html lang="ja">
  <head>
    <meta charset="utf-8">
    <title>tweet</title>
    <script src="//ajax.googleapis.com/ajax/libs/jquery/2.1.4/jquery.min.js"></script>
    <script src="http://{{socketio_host}}/socket.io/socket.io.js"></script>
    <script>
      $(function() {
        var socket = io.connect("http://{{socketio_host}}");
        socket.on("connect", function() {
          console.log("Connected!");
        });
        socket.on("tweet", function(tweet){
          console.log(tweet);
          $("#tweet").prepend(tweet.text + "<br>");
        });
      });
    </script>
  </head>
  <body>
    <div id="tweet"></div>
  </body>
</html>
```

このindex.htmlは[Handlebars.js](http://handlebarsjs.com/)のテンプレートです。`{{socketio_host}}`は`.env`に定義した環境変数を[dotenv](https://www.npmjs.com/package/dotenv)を使ってロードした値が入っています。

## プログラムの実行

Docker Composeからhapiサービスをupします。ブラウザから`.env`ファイルに定義したIPアドレスとポートに接続すると`connected!`のログが出力されます。

```bash
$ docker-compose up
Recreating dockerhapisocketio_hapi_1...
Attaching to dockerhapisocketio_hapi_1
hapi_1 |
hapi_1 | > docker-hapi-socketio@0.0.1 start /app
hapi_1 | > node app.js
hapi_1 |
hapi_1 | connected!
```

サーバーサイドでは5秒間隔でメッセージをemitしています。

```js ~/node_apps/docker-hapi-socketio/app.js
    var tweet = {text: 'hello world!'};
    var interval = setInterval(function () {
        socket.emit('tweet', tweet);
    }, 5000);
```

ブラウザサイドにも5秒間隔でメッセージがappendされていきます。

![hello-socketio.png](/2015/06/14/hapijs-socketio/hello-socketio.png)
