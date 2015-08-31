title: "Framework7でHTML5モバイルアプリをつくる - Part1: Geeting Started"
date: 2015-08-28 09:27:47
tags:
 - Framework7
 - HTML5モバイルアプリ
 - iOS
 - MaterialDesign
 - Express
 - Bower
 - iojs
description: HTML5モバイルアプリ開発のためのFramework7を試してみようと思います。最近はWindows 10もタブレットモードで操作することが多くなりました。WebコンテンツやアプリのUIはモバイル用の方が使いやすく感じるようになってきています。少し調べたところFramework7がネイティブに近い画面をWebで作ることができそうなので、Geeting Startedしてみます。
---

HTML5モバイルアプリ開発のための[Framework7](http://www.idangero.us/framework7/)を試してみようと思います。最近はWindows 10もタブレットモードで操作することが多くなりました。WebコンテンツやアプリのUIはモバイル用の方が使いやすく感じるようになってきています。[少し調べた](/2015/08/27/html5-mobile-apps-resource/)ところFramework7がネイティブに近い画面をWebで作ることができそうなので、Geeting Startedしてみます。


<!-- more -->


## Framework7

[Framework7](http://www.idangero.us/framework7/)はiOSとAndroidのネイティブ風な画面を作ることができる、HTML/JavaScript/CSSのモバイルWebフレームワークです。iOSとMaterialのテーマが使えます。[ドキュメント](http://www.idangero.us/framework7/docs/)や[チュートリアル](http://www.idangero.us/framework7/tutorials/)、[サンプル](http://www.idangero.us/framework7/examples/)も多くそろっているので学習しやすいです。また、ネイティブアプリを作成する前のプロトタイプにも向いているようです。


## プロジェクト

[Getting Started](http://www.idangero.us/framework7/get-started/)の短いチュートリアルを実行するプロジェクトを作成します。リポジトリは[こちら](https://github.com/masato/docker-framework7)です。

基本的には掲載されているコードを少し修正して、[Express](http://expressjs.com/)とDockerで動くようにしただけです。

```bash
$ cd ~/node_apps/docker-framework7
$ tree
.
├── Dockerfile
├── README.md
├── app.js
├── bower.json
├── bower_components -> /dist/bower_components
├── docker-compose.yml
├── node_modules -> /dist/node_modules
├── package.json
└── public
    ├── about.html
    ├── css
    ├── index.html
    └── js
        └── index.js
```

## Dockerfile

ベースイメージに[io.js](https://hub.docker.com/_/iojs/)の[3.2](https://github.com/nodejs/docker-iojs/blob/68a13ab0f190d079b484c67b9eb266ce999420b0/3.2/Dockerfile)を使います。

```bash ~/node_apps/docker-framework7/Dockerfile
FROM iojs:3.2
MAINTAINER Masato Shimizu <ma6ato@gmail.com>

RUN mkdir -p /app
WORKDIR /app

RUN adduser --disabled-password --gecos '' --uid 1000 docker && \
    mkdir -p /dist/node_modules /dist/bower_components && \
    ln -s /dist/node_modules /app/node_modules && \
    ln -s /dist/bower_components /app/bower_components && \
    chown -R docker:docker /app /dist/node_modules /dist/bower_components

USER docker
COPY package.json /app/
COPY bower.json /app/

RUN  npm install

COPY . /app
CMD ["npm","start"]
```

### docker作業ユーザー

npmやbowerはrootで実行しないほうが良いので作業用の「docker」ユーザーを作りました。Dockerホストでコンテナを起動するユーザーと同じ`uid`を指定しています。Dockerホストのカレントディレクトリをコンテナのボリュームにマウントしたときに、直接Dockerホスト上からファイルを修正できるようにしています。


### シンボリックリンク

Dockerfile内で`node_modules`と`bower_components`はシンボリックリンクを作成して、カレントディレクトリには直接インストールしないようにしています。Dockerホストでも同様にシンボリックリンクを作成してボリュームにマウントしたときにこの2つのディレクトリが隠れないようにします。


```bash
$ cd ~/node_apps/docker-framework7
$ ln -s /dist/node_modules .
$ ln -s /dist/bower_components .
```

### docker-compose.yml

docker-compose.ymlは、Dockerホストのカレントディレクトリをコンテナのボリュームにマントしています。またDockerホストの`/etc/localtime`もマウントして、コンテナでも同じタイムゾーンが使えるようにします。

```yml ~/node_apps/docker-framework7/docker-compose.yml
framework7:
  build: .
  volumes:
    - .:/app
    - /etc/localtime:/etc/localtime:ro
  ports:
    - 3033:3000
```

## サーバーサイド

### package.json

`dependencies`フィールドにExpressと[Bower](http://bower.io/)のパッケージを指定します。`scripts`の`start`フィールドに書いた`node app.js`のコマンドは、Dockerfileの`CMD ["npm","start"]`から実行されます。`postinstall`フィールドの`bower install`コマンドは、Dockerfileの`RUN  npm install`でサーバーサイドのnpmパッケージをインストールした後に、Bowerでフロントエンドのパッケージをインストールします。

```json ~/node_apps/docker-framework7/package.json
{
    "name": "framework7-docker",
    "description": "framework7 docker",
    "version": "0.0.1",
    "private": true,
    "dependencies": {
        "express": "~4.13.3",
        "bower": "~1.5.2"
    },
    "scripts": {
        "start": "node app.js",
        "postinstall": "bower install"
    }
}
```

### app.js

今回のExpressは、`/public`と`/bower_components`ディレクトリから静的ファイルをサーブする目的で使います。

```js ~/node_apps/docker-framework7/app.js
"use strict";

var express = require('express'),
    app = express();

app.use(express.static(__dirname + '/public'));
app.use('/bower_components',  express.static(__dirname + '/bower_components'));

var server = app.listen(3000, function () {
  var host = server.address().address,
      port = server.address().port;

  console.log('Example app listening at http://%s:%s', host, port);
});
```

## フロントエンド

### bower.json

[Bower](http://bower.io/)を使って[framework7](https://github.com/nolimits4web/Framework7)のパッケージをインストールします。

```json ~/node_apps/docker-framework7/bower.json
{
    "name": "framework7-docker",
    "version": "0.0.1",
    "private": true,
    "dependencies": {
        "framework7": "~1.2.0"
    }
}
```

### index.html

Getting Startedなので`/bower_components`でインストールしたパッケージを直接参照していますが、gulpからwebpackでビルドしておいた方がよいです。publicディレクトリに配置したフロントエンドのメインプログラムを指定します。

```html ~/node_apps/docker-framework7/public/index.html
    <!-- Path to Framework7 iOS CSS theme styles-->
    <link rel="stylesheet" href="/bower_components/framework7/dist/css/framework7.ios.min.css">
    <!-- Path to Framework7 iOS related color styles -->
    <link rel="stylesheet" href="/bower_components/framework7/dist/css/framework7.ios.colors.min.css">
    <!-- Path to your custom app styles-->
    <!--
    <link rel="stylesheet" href="path/to/my-app.css">
    -->
...
            <div class="page-content">
              <p>Page content goes here</p>
              <!-- Link to another page -->
              <a href="/about.html">About app</a>
            </div>
...
    <!-- Path to Framework7 Library JS-->
    <script type="text/javascript" src="/bower_components/framework7/dist/js/framework7.min.js"></script>
    <!-- Path to your app js-->
    <script type="text/javascript" src="/js/index.js"></script>
```


### index.js

index.jsは[掲載](http://www.idangero.us/framework7/get-started/)されているコードをそのまま使います。`Option 2`をどちらも書いているので、`about`ページに移動したときに'Here comes About page`のメッセージが2度表示されます。

```js ~/node_apps/docker-framework7/public/js/index.js
// Initialize app and store it to myApp variable for futher access to its methods
var myApp = new Framework7();

// We need to use custom DOM library, let's save it to $$ variable:
var $$ = Dom7;

// Add view
var mainView = myApp.addView('.view-main', {
  // Because we want to use dynamic navbar, we need to enable it for this view:
  dynamicNavbar: true
});

// Now we need to run the code that will be executed only for About page.

// Option 1. Using page callback for page (for "about" page in this case) (recommended way):
myApp.onPageInit('about', function (page) {
  // Do something here for "about" page

});

// Option 2. Using one 'pageInit' event handler for all pages:
$$(document).on('pageInit', function (e) {
  // Get page data from event data
  var page = e.detail.page;

  if (page.name === 'about') {
    // Following code will be executed for page with data-page attribute equal to "about"
    myApp.alert('Here comes About page');
  }
});

// Option 2. Using live 'pageInit' event handlers for each page
$$(document).on('pageInit', '.page[data-page="about"]', function (e) {
  // Following code will be executed for page with data-page attribute equal to "about"
  myApp.alert('Here comes About page');
});
```

## 実行とiOSから確認

docker-composeからDockerイメージのビルドとコンテナの起動を行います。

```bash
$ docker-compose build
$ docker-compose up framework7
```

iOSのSafariからdocker-compose.ymlに指定したDockerホストの`3033`ポートを開きます。`/public/js/index.js`のコードは`About app`をクリックした後の`about.html`で実行されます。

![framework7-index.png](/2015/08/28/framework7-getting-started/framework7-index.png)

ページのロード時に`myApp.alert('Here comes About page');`が2回実行されます。

![framework7-about.png](/2015/08/28/framework7-getting-started/framework7-about.png)