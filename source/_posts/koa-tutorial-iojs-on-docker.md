title: "Koa入門 - Part1: io.js on Docker"
date: 2015-06-25 19:55:01
tags:
 - Nodejs
 - ES6
 - Koa
 - iojs
 - Express
 - DockerCompose
description: Node.jsでAPIを作るときのフレームワークに何が良いかいろいろと試しています。Hapi.js、LoopBack、Salis.jsと試してみました。Koaは以前から気になっていましたが、ちょっと難しい印象があって避けていました。しかしES6ことECMAScript 2015がいよいよ承認されました。もたもたしているとES7やES8もすぐに来てしまいます。もうクライアント側もサーバー側もES6に移行していかないといけないです。
---

Node.jsでAPIを作るときのフレームワークに何が良いかいろいろと試しています。[Hapi.js](http://hapijs.com/)、[LoopBack](http://loopback.io/)、[Salis.js](http://sailsjs.org/)、[actionhero.js](http://www.actionherojs.com/)と試してみました。[Koa](http://koajs.com/)は以前から気になっていましたが、ちょっと難しい印象があって避けていました。しかしES6ことECMAScript 2015がいよいよ[承認されました](http://www.ecma-international.org/publications/standards/Ecma-262.htm)。もたもたしているとES7やES8が来てしますので、もうクライアント側もサーバー側もES6に移行していかないといけないです。

## ES6

ES6について全くの初心者なのですが、幸せになれそうな予感がするのでベットしようと思います。

### ES6とio.js

サーバーサイドでES6やKoaを使う場合、[io.js](https://iojs.org/)の方が対応が進んでいます。そもそもio.jsが設立した原因の一つにNode.jsのV8とES6対応の開発方針への不満があったようです。ただ日本のNode.jsの広まりを考えるとサーバーサイドの開発言語としての採用に混乱を招いているようであまり良いことではありません。もっともio.jsがNode Foundationへの参加を決定して今後統合されていくみたいですが。

### ES6と関数型プログラミング

言語仕様として`let`や`const`などで変数のスコープが安全に書けたり、パターンマッチングによる分配束縛(Destructuring)も使えます。

ようやく[async](https://github.com/caolan/async)と`if (error) return callback(error)`のような[Error-First Callback](http://thenodeway.io/posts/understanding-error-first-callbacks/)に慣れてきた程度なのですが、ES6だともっとエレガントににコールバック地獄から抜けられるそうです。[Generator](http://www.ecma-international.org/ecma-262/6.0/#sec-generatorfunction-objects)や[Promise](http://www.ecma-international.org/ecma-262/6.0/#sec-promise-objects)などまだ理解が必要なことが多いです。

またlodashを覚えてからNode.jsでも関数型プログラミング風に書けて読みやすくなりました。ES6では[Arrow Functions](http://www.ecma-international.org/ecma-262/6.0/#sec-arrow-function-definitions)が標準で使えるので無名関数がもっと短く書けるようになります。

Destructuring、Arrow Function、Generator、Promiseを日本語に訳したりカタカナで表記するのも適切か悩むようになりました。明治時代の新漢語みたいにセンスある日本語ができれば良いのですが、そういう時代でもないですし。

## Dockerでio.js環境をつくる

Dockerにはio.jsの[iojs](https://registry.hub.docker.com/_/iojs/)のオフィシャルイメージがあります。GitHubのリポジトリは[docker-iojs](https://github.com/nodejs/docker-iojs)です。バージョンは6月にリリースされたばかりの2.3.0も用意されています。

Node.jsの[オフィシャルイメージ](https://registry.hub.docker.com/_/node/)からONBUILD版を使うときは以下のように書いていました。

```bash Dockerfile
FROM node:0.12-onbuild  
EXPOSE 3000
```

io.jsでも同じようにONBUILD版があります。

```bash Dockerfile
FROM iojs:2.3-onbuild
EXPOSE 3000
```

## ExpressでHello World

io.jsでKoaのプログラムを書く前に、お約束のExpressでHello Worldを動かしてみます。

### プロジェクト

プロジェクトのディレクトリは以下のようになります。

```bash
$ cd ~/node_apps/docker_iojs
$ tree
.
├── Dockerfile
├── app.js
├── docker-compose.yml
└── package.json
```

app.jsはExpressの[Hello World](http://expressjs.com/starter/hello-world.html)を使います。


```js app.js
var express = require('express');
var app = express();

app.get('/', function (req, res) {
  res.send('Hello World!');
});

var server = app.listen(3000, function () {

  var host = server.address().address;
  var port = server.address().port;

  console.log('Example app listening at http://%s:%s', host, port);

});
```

```json package.json
{
    "name": "iojs-express",
    "description": "iojs-express",
    "version": "0.0.1",
    "private": true,
    "dependencies": {
        "express": "^4.13.0"
    },
    "scripts": {
        "start": "node app.js"
    }
}
```

Dockerfileのベースイメージは[iojs:2.3-onbuild](https://github.com/nodejs/docker-iojs/blob/master/2.3/onbuild/Dockerfile)を使います。

```bash Dockerfile
FROM iojs:2.3-onbuild
MAINTAINER Masato Shimizu <ma6ato@gmail.com>
EXPOSE 3000
```

docker-compose.ymlを記述します。ONBUILDを使っているのでカレントディレクトリはコンテナにマップしません。マウントするとカレントディレクトリにはDockerfileで`npm install`した`node_modules`が隠れてしまいます。

```yaml docker-compose.yml
express:
  build: .
  volumes:
    - /etc/localtime:/etc/localtime:ro
  ports:
    - "3030:3000"
```

### 実行

Dockerイメージをビルドしてコンテナを起動します。io.jsでもコマンドはnodeコマンドです。バージョンは`2.3.0`です。Node.jsの方は同じnodeコマンドでも`0.12.4`なので混乱しそうです。

```bash
$ docker-compose build
$ docker-compose up
Creating dockeriojs_express_1...
Attaching to dockeriojs_express_1
express_1 | npm info it worked if it ends with ok
express_1 | npm info using npm@2.11.1
express_1 | npm info using node@v2.3.0
express_1 | npm info prestart iojs-express@0.0.1
express_1 | npm info start iojs-express@0.0.1
express_1 |
express_1 | > iojs-express@0.0.1 start /usr/src/app
express_1 | > node app.js
express_1 |
express_1 | Example app listening at http://:::3000
```

psで見るとDockerホストの3030ポートにマップされています。

```bash
docker-compose ps
        Name            Command    State           Ports
-----------------------------------------------------------------
dockeriojs_express_1   npm start   Up      0.0.0.0:3030->3000/tcp
```

Docckerホストからcurlコマンドを使ってExpressの起動が確認できました。

```bash
$ curl localhost:3030
Hello World!
```



