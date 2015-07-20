title: "ES6で書くIsomorphicアプリ入門 - Part3: React Isomorphic Demo"
date: 2015-07-08 09:22:18
tags:
 - ES6
 - Isomorphic
 - iojs
 - React
 - Express
description: ES6で書くIsomorphicアプリのBoilerplateを調べました。いくつか手を動かしながら勉強していこうと思います。最初はなるべくシンプルなIsomorphicな動作を選びたいのですが、Reactアプリは複雑になりがちで周辺ツールも多くどの構成を選んだら良いか悩みます。Tutorial Setting Up a Simple Isomorphic React appのポストがとてもわかりやすいので写経していきます。
---

ES6で書くIsomorphicアプリのBoilerplateを[調べました](/2015/07/02/iojs-es6-isomorphic-boilerplate/)。いくつか手を動かしながら勉強していこうと思います。最初はなるべくシンプルなIsomorphicな動作を選びたいのですが、Reactアプリは複雑になりがちで周辺ツールも多くどの構成を選んだら良いか悩みます。[Tutorial: Setting Up a Simple Isomorphic React app](http://jmfurlott.com/tutorial-setting-up-a-simple-isomorphic-react-app/)のポストがとてもわかりやすいので写経していきます。


## Isomorphicな特徴

IsomorphicなReactアプリはクライアントのコードは1つのファイルにバンドルして、HTMLからロードします。最初にHTMLを開いたときはサーバーサイドでレンダリングされたコンポーネントを表示しますが、バンドルされたReactアプリがロードされたら上書きします。

## プロジェクト

[Tutorial: Setting Up a Simple Isomorphic React app](http://jmfurlott.com/tutorial-setting-up-a-simple-isomorphic-react-app/)のリポジトリは[react-isomorphic-boilerplate](https://github.com/jmfurlott/react-isomorphic-boilerplate)です。

今回作成したディレクトリ構造です。作業リポジトリは[こちら](https://github.com/masato/react-isomorphic-boilerplate-app)です。

```bash
$ cd ~/node_apps/react-isomorphic-boilerplate-app
$ tree
.
├── Dockerfile
├── README.md
├── css
├── docker-compose.yml
├── node_modules -> /dist/node_modules
├── package.json
├── src
│  ├── client
│  │   └── entry.js
│  ├── server
│  │   ├── server.js
│  │   └── webpack.js
│  └── shared
│      ├── components
│      │  └── AppHandler.js
│      └── routes.js
├── views
│  └── index.jade
└── webpack.config.dev.js
```

### Dockerfile

io.jsのベースイメージを使いDockerfileを用意します。

```bash ~/node_apps/react-isomorphic-boilerplate-app/Dockerfile
FROM iojs:2.3
MAINTAINER Masato Shimizu <ma6ato@gmail.com>

RUN mkdir -p /app
WORKDIR /app

RUN adduser --disabled-password --gecos '' --uid 1000 docker && \
    mkdir -p /dist/node_modules && \
    ln -s /dist/node_modules /app/node_modules && \
    chown -R docker:docker /app /dist/node_modules

USER docker
COPY package.json /app/
RUN  npm install

COPY . /app
CMD ["npm","start"]
```

### docker-compose.yml

docker-compose.ymlには環境変数としてpublic ip addressを指定します。HTMLからWebpack dev serverに接続するためのIPアドレスです。サーバーサイドはクラウド上で動作しているため、ブラウザからはリモートでアクセスする必要があります。PUBLIC_IPにインターネットから接続できるIPアドレスを指定します。

```yml ~/node_apps/react-isomorphic-boilerplate-app/docker-compose.yml
npm:
  build: .
  volumes:
    - .:/app
    - /etc/localtime:/etc/localtime:ro
  environment:
    - PUBLIC_IP=xxx.xxx.xxx.xxx
    - EXPRESS_PORT=3030
    - WEBPACK_PORT=8090
  ports:
    - 3030:3030
    - 8090:8090
```

docker-composeコマンドを短縮するためにエイリアスを用意します。

```bash ~/.bashrc
alias iojs-run='docker-compose run --rm npm'
alias iojs-up='docker-compose up npm'
```

package.jsonはデフォルトで最小限を書いておきます。

```json ~/node_apps/react-isomorphic-boilerplate-app/package.json
{
    "name": "react-isomorphic",
    "description": "react-isomorphic",
    "version": "0.0.1",
    "private": true
}
```

docker-composeのエイリアスを使って必要なモジュールをnpmでインストールします。

```bash
$ iojs-run npm install --save-dev react webpack react-router react-hot-loader webpack-dev-server babel-loader
```

用意しておいたpackage.jsonに`devDependencies`のセクションが追加されました。

```json ~/node_apps/react-isomorphic-boilerplate-app/package.json
{
    "name": "react-isomorphic",
    "description": "react-isomorphic",
    "version": "0.0.1",
    "private": true,
    "devDependencies": {
        "babel": "^5.6.14",
        "babel-core": "^5.6.17",
        "babel-loader": "^5.3.1",
        "express": "^4.13.1",
        "jade": "^1.11.0",
        "node-libs-browser": "^0.5.2",
        "nodemon": "^1.3.7",
        "react": "^0.13.3",
        "react-hot-loader": "^1.2.8",
        "react-router": "^0.13.3",
        "webpack": "^1.10.1",
        "webpack-dev-server": "^1.10.1"
    },
    "scripts": {
        "clean": "rm -rf lib",
        "watch-js": "babel src -d lib --experimental -w",
        "dev-server": "node lib/server/webpack.js",
        "server": "nodemon lib/server/server.js",
        "start": "npm run watch-js & npm run dev-server & npm run server",
        "build": "npm run clean && babel src -d lib --experimental"
    }
}
```

`scripts`セクションに開発に必要なコマンドを用意します。ごちゃごちゃしているのでGulpでまとめた方が良さそうです。ホットロードもしますが、ES6で書いたコードは`build`でコンパイルしておきます。ブラウザからのリクエストを処理するExpressのサーバーと、バンドルされたReactアプリを返すWebpack dev serverの2つを起動します。

package.jsonが出来上がったのでイメージにビルドします。

```bash
$ docker-compose build
```

## サーバーサイド

プロジェクトにプログラムを配置するディレクトリを作成します。

```bash 
$ mkdir -p src/{server,shared,client} views
```

* server: ExpressとWebpack dev server
* client: Reace bundleのエントリポイント
* shared: componentsとroutes

### views/index.jade

ビューは[Jade](http://jade-lang.com/)のテンプレートエンジンを使います。`#app!= content`の記述で`<div id="app">`要素を作成します。

```jade ~/node_apps/react-isomorphic-boilerplate-app/views/index.jade
html
  head
    title="React Isomorphic App"
    meta(charset='utf-8')
    meta(http-equiv='X-UA-Compatible', content='IE=edge')
    meta(name='description', content='')
    meta(name='viewport', content='width=device-width, initial-scale=1')

  body
    #app!= content
    script(src='http://'+public_ip+':'+webpack_port+'/js/app.js', defer)
```

### src/server/server.js

ExpressのコードもES6で書きます。routeは`/*`で全部拾って`react-router`に渡します。このプロジェクトにはfavicon.icoを用意していませんが、staticなコンテンツもreact-routerにroutingの役割が回ってしまいます。

```bash
Warning: No route matches path "/favicon.ico". Make sure you have <Route path="/favicon.ico"> somewhere in your routes
```

`react-router`はサーバーとクライアントで`shared/routes`を共有しています。デフォルトのサンプルだと同じコンポーネントを使っているので、Expressが``<div id="app">``にrenderしたcontentと、React bundleがロードされた後に、`document.getElementById('app')`でrenderする内容の区別がつきません。Isomorphicではなくなりますが、処理をわかりやすくするためにcontents変数にはコンポーネントではなく、デフォルトの文字列を入れるようにしました。


```js ~/node_apps/react-isomorphic-boilerplate-app/src/server/server.js
import express from 'express';
import React from 'react';
import Router from 'react-router';
const app = express();

// set up Jade
app.set('views', './views');
app.set('view engine', 'jade');

import routes from '../shared/routes';

app.get('/*', function(req, res) {
    Router.run(routes, req.url, Handler => {
        //let content = React.renderToString(<Handler />);
        let content = 'empty!';
        res.render('index', { public_ip_port: process.env.PUBLIC_IP_PORT,
                              content: content });
    });
});

var server = app.listen(process.env.EXPRESS_PORT, function() {
    var host = server.address().address;
    var port = server.address().port;
    console.log('Example app listening at http://%s:%s', host, port);
});
```

### src/server/webpack.js

Reactアプリをレンダーする開発用サーバーです。Node.jsのサーバーとは別に動作します。WebpackDevServerインスタンスは、`webpack.config.dev.js`に書かれた設定を読み込んで使います。今回はサーバーサイドがクラウド上で動作しているため、Webpack dev serverはリモートから接続します。`localhost`でなく`0.0.0.0`でLISTENするように変更しました。

```js ~/node_apps/react-isomorphic-boilerplate-app/src/server/webpack.js

import WebpackDevServer from "webpack-dev-server";
import webpack from "webpack";
import config from "../../webpack.config.dev";

var server = new WebpackDevServer(webpack(config), {
    // webpack-dev-server options
    publicPath: config.output.publicPath,
    hot: true,
    stats: {colors: true},
});

server.listen(process.env.WEBPACK_PORT, "0.0.0.0", function() {});
```

### webpack.config.dev.js

entryにapp.jsにbundleするエントリポイントを複数指定します。

* webpack-dev-serverのホストとポート
* ホットロード
* アプリのクライアント


```js  ~/node_apps/react-isomorphic-boilerplate-app/webpack.config.dev.js
var webpack = require('webpack');

var public_url = 'http://'+process.env.PUBLIC_IP+':'+process.env.WEBPACK_PORT;

module.exports = {
    devtool: 'inline-source-map',
    entry: [
        'webpack-dev-server/client?'+public_url,
        'webpack/hot/only-dev-server',
        './src/client/entry',
    ],
    output: {
        path: __dirname + '/public/js/',
        filename: 'app.js',
        publicPath: public_url+'/js/',
    },
    plugins: [
        new webpack.HotModuleReplacementPlugin(),
        new webpack.NoErrorsPlugin(),
    ],
    resolve: {
        extensions: ['', '.js']
    },
    module: {
        loaders: [
            { test: /\.jsx?$/, loaders: ['react-hot', 'babel-loader?experimental'], exclude: /node_modules/ }
        ]
    }
}
```

## 共有

### src/shared/components/AppHandler.js

AppHandler.jsはサーバーとクライアントで共有しているコンポーネントです。今回のテストではExpressからIsomorphicにコンポーネントを共有して、サーバーサイドレンダリングできることを確認した後、静的な文字列に変更しています。

```js ~/node_apps/react-isomorphic-boilerplate-app/src/shared/components/AppHandler.js
import React from 'react';

export default class AppHandler extends React.Component {
    render() {
        return <div>Hello App Handler</div>;
    }
}
```

### src/shared/routes.js

`react-router`のroutesを定義します。Routeはpathの`/`にAppHandlerコンポーネントを表示します。


```js ~/node_apps/react-isomorphic-boilerplate-app/src/shared/routes.js
import { Route } from 'react-router';
import React from 'react';

import AppHandler from './components/AppHandler';

export default (
    <Route handler={ AppHandler } path="/" />
);
```

## クライアント

### src/client/entry.js

React bundleのエントリポイントです。サーバーサイドでは`React.renderToString(<Handler />);`でJadeでレンダリングするcontents変数を作成しましたが、クライアントサイドでは` React.render(<Handler />, document.getElementById('app'));`を使って、直接divのidを指定してコンポーネントをマウントします。

```js ~/node_apps/react-isomorphic-boilerplate-app/src/client/entry.js
import React from 'react';
import Router from 'react-router';
import routes from '../shared/routes';

Router.run(routes, Router.HistroyLocation, (Handler, state) => {
    React.render(<Handler />, document.getElementById('app'));
});
```

## 起動とテスト

一応クリーンビルドしておきます。

```bash
$ iojs-run npm run clean
$ iojs-run npm run build
...
npm info postclean react-isomorphic@0.0.1
npm info ok
src/client/entry.js -> lib/client/entry.js
src/server/server.js -> lib/server/server.js
src/server/webpack.js -> lib/server/webpack.js
src/shared/components/AppHandler.js -> lib/shared/components/AppHandler.js
src/shared/routes.js -> lib/shared/routes.js
npm info postbuild react-isomorphic@0.0.1
npm info ok
```


`docker-compose up npm`のエイリアスである、`iojs-up`を実行します。

```bash
$ iojs-up
...
npm_1 | webpack: bundle is now VALID.
```

ブラウザからDockerホストのパブリックIPアドレスを実行します。接続先はExpressサーバーの3030ポートです。

http://xxx.xxx.xxx.xxx:3030/

本来はIsomorphicに共有しているコンポーネントをサーバーサイドでrenderします。React bundleが上書きする動作を確認するため、'empty!'の静的な文字列が最初にrenderします。index.htmlがロードされたあと、React bundleがWebpack dev serverからロードされます。`react-router`の`/`pathに従いAppHandler.jsがcomponentとして表示されます。


## 課題

シンプルなサンプルなのでとてもわかりやすいですが、Expressのrouteを`/*`としてすべてreact-routerに渡しています。その反面favicon.icoなど静的ファイルもreact-routerでハンドリングが必要になったり、Isomorphicで共有いしているコンポーネントの動作が見えづらいところがあります。