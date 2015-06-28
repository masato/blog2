title: "Koa入門 - Part4: ES6でIsomorphicなDeku"
date: 2015-06-28 03:12:14
tags:
 - Nodejs
 - iojs
 - ES6
 - Koa
 - Babel
description: Reactのリアクティブとコンポーネント指向の考えはすごく良いのですが、どうも書き方に馴染めません。IsomorphicもJavaScriptで書くよりClojureで書く方が楽しいのでReagentを使っています。Node.jsのES6時代に向けたKoaのIsomorphicを調べていると、horseやDekuといったライブラリがありました。今回はDekuの方を試してみます。
---

[React](http://facebook.github.io/react/)のリアクティブとコンポーネント指向の考えはすごく良いのですが、どうも書き方に馴染めません。IsomorphicもJavaScriptで書くよりClojureで書く方が楽しいので[Reagent](http://holmsand.github.io/reagent/)を使っています。Node.jsのES6時代に向けたKoaのIsomorphicを調べていると、[horse](https://github.com/reddit/horse)や[Deku](https://github.com/dekujs/deku)といったライブラリがありました。今回はDekuの方を試してみます。

<!-- more -->

## Deku

[Deku](https://github.com/dekujs/deku)はReactのalternativeとしてUIのコンポーネントを作成するライブラリです。

### NIHではない

なぜDekuを開発し始めたかは[Deku: How we built our functional alternative to React](https://segment.com/blog/deku-our-functional-alternative-to-react/)に書いてあります。頭が痛いすぐ内製でやりたがる[NIH](https://ja.wikipedia.org/wiki/NIH%E7%97%87%E5%80%99%E7%BE%A4)ではないと言っています。彼らはフロントエンドのパッケージマネージャとして[Duo](http://duojs.org/)を使っているようです。Reactのような大きなブラックボックス中でたくさんの機能を実装するよりも、小さなモジュールで構成した方が見通しもよく、デバッグしやすいといった考え方が背景にあります。見通しがよく何か問題があったときに原因を特定しやすいというのはフレームワークを選ぶ上でとても重要なことです。

### IsomorphicかAPIサーバーか

以前[Reagent入門 - Part3: クライアントとサーバーの通信パターン](/2015/06/19/reagent-client-server-connection/)や[Reagent入門 - Part4: SPAとフォームの要件](/2015/06/22/reagent-form-bootstrap/)で考察しました。今のところSPAはIsomorphicでなくAPIサーバーにして、毎回面倒ですがルーティングやモデルはクライアントもサーバーも2回書くのがよいと思っています。

Isomorphic初心者なので、Clojure/ClojureScriptで書いていてもサーバーのコードを書いているのかクライアントなのか時々わからなくなります。ルーティングを書いているときなどほとんど同じコードなので混乱してしまいます。

## プロジェクト

今回のプロジェクトは以下の構成です。[Deku](https://github.com/dekujs/deku)とサンプルの[todomvc](https://github.com/dekujs/todomvc)を読みながらインストールとKoaを使ったサーバサイドレンダリングを試してみます。


## Dekuのビルド方法

[Deku](https://github.com/dekujs/deku)はクライアントサイドのライブラリです。ES6で書くのでいずれにいても[Babel](https://babeljs.io/)は必要です。

### Browserfify

[Browserfify](http://browserify.org/)を使うのが、サーバーサイドとクライアントサイドのどちらのレンダリングを使う場合でも一番簡単なようです。babelifyを使うとJSXのコンパイルも簡単にできます。

```basg
$ browserify -t babelify main.js > build.js
```

### Duo

[Duo](http://duojs.org/)は次世代のフロントエンドのパッケージマネージャです。DuoからBabelを使うときには[duo-babel](https://github.com/babel/duo-babel)が必要になります。Dekuは[Duo](http://duojs.org/)を推しているようなので使ってみようと思います。

Duoの考え方は1つのパッケージは1つの機能を提供することです。コード上のインポートは直接GitHubのパッケージを指定してます。


```js
import {element,tree,render} from 'segmentio/deku@0.2.1'
```

ES6で書いてBabelが必要になるので、duoコマンドには`--use duo-babel`フラグを付けてコンパイルして使います。

```js
$ duo --use duo-babel main.js > build.js
```

### babel/register

単純なHello Worldなので、今回はindex.jsで`babel/register`のフックからrequireしたapp.es6を自動的にコンパイルします。本来のIsomorphicならばクライアント側のES6コードは`browserify`でコンパイルして、ES5
にビルド済みのjsファイルをindex.htmlなどから`<script src="/build/index.js"></script>`として最初にロードする必要があります。次回はクライアント側の`browserify`を試してみます。

## プロジェクト

適当なディレクトリにプロジェクトを作成します。リポジトリは[こちら](https://github.com/masato/docker-deku-koa)です。

```bash
$ cd ~/node_apps/docker-deku-koa
$ tree
.
├── Dockerfile
├── README.md
├── client
│   └── helloworld.es6
├── docker-compose.yml
├── node_modules -> /dist/node_modules
├── package.json
└── server
    ├── app.es6
    └── index.js
```

### Dockerfileとdocker-compose.yml

Dockerfileのベースイメージは[iojs](https://registry.hub.docker.com/_/iojs/)を使います。

```bash ~/node_apps/docker-deku-koa/Dockerfile
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
ENTRYPOINT ["npm", "start"]
CMD []
```

docker-compose.ymlではDockerホストのカレントディレクトリごと、コンテナの`WORKDIR`である`/app`にマウントします。


```yaml ~/node_apps/docker-deku-koa/docker-compose.yml
deku:
  build: .
  volumes:
    - .:/app
    - /etc/localtime:/etc/localtime:ro
  ports:
    - "3030:3000"
```

コンテナにマウントしたときに`node_modules`が隠れないように、カレントディレクトリに予めシムリンクを作っておきます。

```bash
$ ln -s /dist/node_modules .
```

### package.json

package.jsonに必要なパッケージをdependenciesに追加します。

```json ~/node_apps/docker-deku-koa/package.json
{
    "name": "deku-koa-example",
    "description": "deku-koa-example",
    "version": "0.0.1",
    "private": true,
    "dependencies": {
        "deku": "^0.4.5",
        "koa": "^0.21.0",
        "babel": "^5.6.7"
    },
    "scripts": {
        "start": "node server/index.js"
    }
}
```

## サーバーサイド

`babel/register`のフックで`app.es6`をrequireしたときに自動的にES5にコンパイルします。`jsxPragma`オプションに`element`を指定しています。こうするとJSXの変換にBabelがデフォルトのReact.createElementの代わりに、`import {element} from 'deku'`のelementを使うようになります。


```js ~/node_apps/docker-deku-koa/server/index.js
'use strict';

// ES6 support
require('babel/register')({ jsxPragma: 'element' });
require('./app');
```

app.es6はES6で書いたKoaのコードです。IsomorphicなKoaのテストなので、クライアントサイドのコードをimportしてレンダリングしています。

```js ~/node_apps/docker-deku-koa/server/app.es6
'use strict'

import koa from 'koa'
import HelloWorld from '../client/helloworld'
import {element, tree, renderString} from 'deku'

const app = koa()

app.use(function *() {
  this.body = renderString(tree(<HelloWorld text="Hello World!" />))
})

app.listen(3000)
```

## クライアントサイド

`client/helloworld.es6`で、サーバーサイドの`server/app.es6`でも使う`HelloWorld`コンポーネントを定義しています。

```js ~/node_apps/docker-deku-koa/client/helloworld.es6
'use strict'

import {element, render} from 'deku'

var HelloWorld = {
  render(component) {
    let {props,state} = component
    return (
      <div>
        {props.text}
      </div>
    )
  }
}

export default HelloWorld
```

## Dockerイメージのビルドと起動Docker

```bash
$ cd ~/node_apps/docker-deku-koa
$ docker-compose build
$ docker-compose up
Recreating dockerdekukoa_deku_1...
Attaching to dockerdekukoa_deku_1
deku_1 | npm info it worked if it ends with ok
deku_1 | npm info using npm@2.11.1
deku_1 | npm info using node@v2.3.0
deku_1 | npm info prestart deku-koa-example@0.0.1
deku_1 | npm info start deku-koa-example@0.0.1
deku_1 |
deku_1 | > deku-koa-example@0.0.1 start /app
deku_1 | > node server/index.js
deku_1 |
```

Dockerホストの3030ポートにマップされています。curlコマンドでテストするとHello World!のdiv要素が返りました。

```bash
$ curl localhost:3030
<div>Hello World!</div>
```
