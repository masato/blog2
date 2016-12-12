title: "Koa入門 - Part2: babel/registerでES6を自動コンパイル"
date: 2015-06-26 12:58:06
tags:
 - Nodejs
 - ES6
 - Koa
 - iojs
 - Babel
description: BabelはES6のコードをES5にコンパイルしてくれるツールです。以前は6to5と呼ばれていました。ECMAScript 6(ES6)が正式にECMAScript 2015(ES2015)という名前になったのでツール名も変更になったようです。ただ略称の方はES6で定着しているのでこのまま使いそうです。io.jsではKoaのジェネレータ関数などES6の機能を--harmmonyフラグを付けなくても使えます。せっかくなのですべてES6で書きたいところです。io.jsではV8に追加されるES6の機能を積極的にサポートしていますが、全部ではないので現状はBabelでES5に変換する必要があります。
---

[Babel](https://babeljs.io/)はES6のコードをES5にコンパイルしてくれるツールです。以前は6to5と呼ばれていました。ECMAScript 6(ES6)が正式にECMAScript 2015(ES2015)という名前になったのでツール名も変更になったようです。ただ略称の方はES6で定着しているのでこのまま使いそうです。[io.js](https://iojs.org)では[Koa](http://koajs.com/)の[ジェネレーター関数](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Statements/function*)などES6の機能を`--harmmony`フラグを付けなくても使えます。せっかくなのですべてES6で書きたいところです。io.jsではV8に追加されるES6の機能を積極的にサポートしていますが、全部ではないので現状はBabelでES5に変換する必要があります。

<!-- more -->

## プロジェクト

適当なディレクトリにプロジェクトを作成します。リポジトリは[こちら](https://github.com/masato/docker-koa-babel-hook)です。

```bash
$ cd ~/node_apps/docker-koa-babel-hook
$ tree
.
├── Dockerfile
├── app.es6
├── docker-compose.yml
├── index.js
└── package.json
```

Dockerfileのベースイメージはオフィシャルの[io.jsイメージ](https://registry.hub.docker.com/_/iojs/)からONBUILDを使います。

```bash ~/node_apps/docker-koa-babel-hook/Dockerfile
FROM iojs:2.3-onbuild
EXPOSE 3000
```

docker-compose.ymlは特に設定は行わず、Dockerホストで他のサービスが3000ポートを使っている関係から、3030にマップします。

```yaml ~/node_apps/docker-koa-babel-hook/docker-compose.yml
koa:
  build: .
  volumes:
    - /etc/localtime:/etc/localtime:ro
  ports:
    - "3030:3000"
```

package.jsonはdependenciesから[Koa](http://koajs.com/)と[Babel](https://babeljs.io/)をインストールします。

```json ~/node_apps/docker-koa-babel-hook/package.json
{
  "name": "docker-koa-babel-hook",
  "description": "docker-koa-babel-hook",
  "version": "0.0.1",
  "private": true,
  "scripts": {
    "start": "node index.js"
  },
  "dependencies": {
    "koa" : "^0.21.0",
    "babel": "^5.6.14"
  }
}
```

## index.js

エントリポイントのindex.jsに`babel/register`を記述します。[babel/register](https://babeljs.io/docs/usage/require/)を最初に記述してrequire()をフックします。この後のrequire()したファイルはBabelがその場自動的にES6をES5にコンパイルしてくれます。


```js ~/node_apps/docker-koa-babel-hook/index.js
'use strict';

require('babel/register');
require('./app');
```

[require('babel/register') doesn't work](http://stackoverflow.com/questions/29207878/requirebabel-register-doesnt-work)に書き込みがあるように、`babel/register`を記述したファイル自体はコンパイル対象になりません。そのため`app.es6`ではなく、app.es6をrequireするindex.jsの先頭に記述します。

## app.es6

app.es6が実際のアプリのコードです。拡張子を`es6`にするとEmacs 24のメジャーモードに対応していないので、`es6`の拡張子をinit.elなどに追加します。今回は[init-loader](https://github.com/emacs-jp/init-loader)を使っています。

```el ~/.emacs.d/inits/05-js-mode.el
(add-to-list 'auto-mode-alist '("\\.es6$" . js-mode))
```

app.es6はHelloWorldをするだけの単純なKoaのコードです。`import`や`const`、GeneratorがあるES6のコードです。

```js ~/node_apps/docker-koa-babel-hook/app.es6
'use strict'

import koa from 'koa'

const app = koa()

app.use(function *() {
  this.body = 'Hello World';
});

app.listen(3000);
```

## Dockerイメージのビルドと実行

Dockerイメージをビルドして実行します。

```bash
$ cd ~/node_apps/docker-koa-babel-hook/
$ docker-compose build
$ docker-compose up
```

Dockerホストからcurlを使ってテストします。docker-compose.ymlでDockerホストの3030ポートにマップしています。Hello Worldが表示されました。

```bash
$ curl localhost:3030
Hello World
```
