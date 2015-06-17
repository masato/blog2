title: "React入門 - Part2: Browserify/Reactify/Gulpを使う"
date: 2014-12-27 14:30:04
tags:
 - React
 - Browserify
 - Reactify
 - Gulp
 - npm
 - Nodejs
 - ReactiveProgramming
description: 前回作成したhelloworldはブラウザでJSXTransformerを読み込みオンラインでjsにコンパイルしていました。実行時にChromeのデベロッパーツールに表示されるように、予めJSXはコンパイルすることが推奨されています。コンパイルの方法は、jsxコマンドを使う方法、 BrowserifyとReactifyを使う方法、最後にGulpのタスクでまとめる方法を順番に試してみます。
---

[前回作成したhelloworld](/2014/12/26/react-getting-started/)はブラウザでJSXTransformerを読み込みオンラインでjsにコンパイルしていました。実行時にChromeのデベロッパーツールに表示されるように、予めJSXはコンパイルすることが推奨されています。コンパイルの方法は、jsxコマンドを使う方法、 [Browserify](https://github.com/substack/node-browserify)と[Reactify](https://github.com/andreypopp/reactify)を使う方法、最後に[Gulp](https://github.com/gulpjs/gulp/)のタスクでまとめる方法を順番に試してみます。

<!-- more -->

### Getting Startedの復習

作成したhelloworld.jsxを少し修正してコンポーネントを作り構成してみます。ディレクトリ構成は以下です。

``` bash
$ cd ~/react_apps/helloworld/
$ mkdir src dist
$ mv helloworld.jsx src/
$ tree .
.
|-- dist
|-- index.html
`-- src
    `-- helloworld.jsx
```

Nameコンポーネントは作成時にプロパティを指定しています。nameプロパティは`{this.props.name}`として参照できます。

``` jsx ~/react_apps/helloworld/src/helloworld.jsx
var Name = React.createClass({
  render: function() {
    return (
      <span>{this.props.name}</span>
    );
  }
});
var HelloWorld = React.createClass({
  render: function() {
    return (
      <div>
        <h1>Hello, world!</h1>
        <Name name="Masato" />
      </div>
    );
  }
});
React.render(
  <HelloWorld />,
  document.getElementById('example')
);
```

jsxファイルのパスも変更します。

``` html ~/react_apps/helloworld/index.html
<!DOCTYPE html>
<html>
  <head>
    <title>Hello World</title>
    <script src="http://fb.me/react-0.12.2.js"></script>
    <script src="http://fb.me/JSXTransformer-0.12.2.js"></script>
    <script type="text/jsx" src="./src/helloworld.jsx"></script>
  </head>
  <body>
    <div id="example"></div>
  </body>
</html>
```

SimpleHTTPServerを起動します。

``` bash
$ python -m SimpleHTTPServer 8080
```

Dockerコンテナで作業しているので、ngrokでトンネルします。

``` bash
$ docker run -it --rm wizardapps/ngrok:latest ngrok 172.17.1.157:8080
```

ブラウザから確認します。
http://684baaf0.ngrok.com


### jsxコマンドを使う

Browserfyを使う前に、[Getting Started](http://facebook.github.io/react/docs/getting-started.html)に書いてあるjsxコマンドを試してみます。react-toolsをnpmでインストールします。

``` bash
$ npm install -g react-tools
```

jsxコマンドでsrcディレクトリにあるjsxファイルを、buildディレクトリにJavaScriptとしてコンパイルします。ファイルの拡張子をjsxにしているので、-xフラグの指定が必要です。

``` bash
$ cd ~/react_apps/helloworld/
$ jsx -x jsx src/ dist/
built Module("helloworld")
["helloworld"]
```

jsファイルのロードをbodyに移動します。JSXTransformerも予めjsにコンパイルして不要になったので削除します。

``` html ~/react_apps/helloworld/index.html
<!DOCTYPE html>
<html>
  <head>
    <title>Hello World</title>
    <script src="http://fb.me/react-0.12.2.js"></script>
  </head>
  <body>
    <div id="example"></div>
    <script src="./dist/helloworld.js"></script>
  </body>
</html>
```

ngrok経由でトンネルして、ブラウザで確認します。
http://684baaf0.ngrok.com/

### BrowserifyとReactifyを使う

[Browserify](https://github.com/substack/node-browserify)ブラウザ上でもNode.js用モジュールを使えるようにするライブラリです。[Reactify](https://github.com/andreypopp/reactify)はJSX用のtransform moduleです。npmコマンドを使い、プロジェクトのディレクトリにBrowserify と Reactifyをインストールします。

``` bash
$ cd ~/react_apps/helloworld
$ npm install react browserify reactify
```

jsxファイルでNode.jsと同じようにreactモジュールをrequireします。

``` jsx ~/react_apps/helloworld/src/helloworld.jsx
var React = require('react');
var Name = React.createClass({
  render: function() {
    return (
      <span>{this.props.name}</span>
    );
  }
});
var HelloWorld = React.createClass({
  render: function() {
    return (
      <div>
        <h1>Hello, world!</h1>
        <Name name="Masato" />
      </div>
    );
  }
});
React.render(
  <HelloWorld />,
  document.getElementById('example')
);
```

requireでモジュールを読み込んでいるので、index.htmlファイルからreact.jsのロードを削除します。

``` html ~/react_apps/helloworld/index.html
<!DOCTYPE html>
<html>
  <head>
    <title>Hello World</title>
  </head>
  <body>
    <div id="example"></div>
    <script src="./dist/helloworld.js"></script>
  </body>
</html>
```

browserifyコマンドに-tフラグでreactifyモジュールを指定し、jsxファイルをjsにコンパイルします。生成されたhelloworld.jsをscript要素でロードすると、ブラウザでもNode.jsのモジュールが使えるようになります。コンパイルされたjsファイルはrequireしたreactモジュールもまとめて1つのファイルになっています。

``` bash
$ browserify -t reactify src/helloworld.jsx > dist/helloworld.js
```

ngrok経由でトンネルして、ブラウザで確認します。
http://684baaf0.ngrok.com/

### Gulpを使う

[Gulp](https://github.com/gulpjs/gulp/)は手続き型で設定ファイルを記述する、タスクランナー＆ビルドツールです。Gulpを使うため、少しリファクタリングします。

``` bash
$ cd ~/react_apps/helloworld-gulp
$ tree .
.
|-- dist
|-- gulpfile.js
|-- index.html
`-- src
    |-- main.js
    `-- view.jsx
```

npmコマンドでGulpをグローバルにインストールします。

``` bash
$ npm install -g gulp
```

npmコマンドでモジュールをローカルにインストールします。gulpはローカルのプロジェクトにインストールされたモジュールを使います。

``` bash
$ npm install gulp react browserify reactify vinyl-source-stream
```

gulpfile.jsを作成します。reactifyでtransformしたjsファイルは、`./dist/app.js`に出力されます。

``` js ~/react_apps/helloworld-gulp/gulpfile.js
var gulp = require('gulp');
var browserify = require('browserify');
var source = require("vinyl-source-stream");
var reactify = require('reactify');

gulp.task('browserify', function(){
  var b = browserify({
    entries: ['./src/main.js'],
    transform: [reactify]
  });
  return b.bundle()
    .pipe(source('app.js'))
    .pipe(gulp.dest('./dist'));
});
```

helloworld.jsxファイルをリファクタリングして、main.jsとview.jsxファイルに分割します。

``` js ~/react_apps/helloworld-gulp/src/main.js
var React = require('react');

var view = require('./view.jsx');
React.renderComponent(
  view(),
  document.getElementById('content')
);
```

view.jsxは、HelloWorldコンポーネントを`module.exports`に代入して公開して、main.jsから参照できるようにします。

``` js ~/react_apps/helloworld-gulp/src/view.jsx
var React = require('react');
var Name = React.createClass({
  render: function() {
    return (
      <span>{this.props.name}</span>
    );
  }
});
var HelloWorld = React.createClass({
  render: function() {
    return (
      <div>
        <h1>Hello, world!</h1>
        <Name name="Masato" />
      </div>
    );
  }
});
module.exports = HelloWorld;
```

index.htmlでは、`gulp browserify`タスクで1つにコンパイルした`./dist/app.js`ファイルをロードします。

``` html ~/react_apps/helloworld-gulp/index.html
<!DOCTYPE html>
<html>
  <head>
    <title>Hello World</title>
  </head>
  <body>
    <div id="content"></div>
    <script src="./dist/app.js"></script>
  </body>
</html>
```

`gulp browserify`タスクを実行してJSXのコンパイルとrequireしたモジュールなどをまとめます。

``` bash
$ gulp browserify
[13:47:57] Using gulpfile ~/react_apps/helloworld-gulp/gulpfile.js
[13:47:57] Starting 'browserify'...
[13:47:59] Finished 'browserify' after 2.03 s
```

ngrok経由でトンネルして、ブラウザで確認します。
http://684baaf0.ngrok.com/

### package.jsonを作成する

新しい環境でも同じモジュールがインストールできるように、package.jsonを作成してプロジェクトの情報を記述しておきます。最初にマニュアルでインストールした`node_modules`を削除します。

``` bash
$ cd ~/react_apps/helloworld-gulp
$ rm -fr node_modules/
```

`npm init`コマンドを実行して、対話的にpackage.jsonを作成します。GitHubのリポジトリも自動で入れてくるので、入力した値はauthorだけでpackage.jsonファイルを作成できました。

``` bash
$ npm init
This utility will walk you through creating a package.json file.
It only covers the most common items, and tries to guess sane defaults.

See `npm help json` for definitive documentation on these fields
and exactly what they do.

Use `npm install <pkg> --save` afterwards to install a package and
save it as a dependency in the package.json file.

Press ^C at any time to quit.
name: (helloworld-gulp)
version: (1.0.0)
description:
entry point:
test command:
git repository: (https://github.com/masato/helloworld-gulp.git)
keywords:
author: Masato Shimizu
license: (ISC)
About to write to /home/docker/react_apps/helloworld-gulp/package.json:

{
  "name": "helloworld-gulp",
  "version": "1.0.0",
  "description": "",
  "main": "",
  "scripts": {
    "test": "echo \"Error: no test specified\" && exit 1"
  },
  "repository": {
    "type": "git",
    "url": "https://github.com/masato/helloworld-gulp.git"
  },
  "author": "Masato Shimizu",
  "license": "ISC",
  "bugs": {
    "url": "https://github.com/masato/helloworld-gulp/issues"
  },
  "homepage": "https://github.com/masato/helloworld-gulp"
}


Is this ok? (yes) yes
```

前回は`--save-dev`フラグを付けませんでしたが、package.jsonのdevDependenciesに設定が書かれるように、npm installするときは`--save-dev`を忘れずつけます。

``` bash
$ npm install --save-dev gulp react browserify reactify vinyl-source-stream
```

package.jsonを確認すると、devDependenciesが追加されました。

``` json ~/react_apps/helloworld-gulp/package.json
...
  "devDependencies": {
    "browserify": "^8.0.2",
    "gulp": "^3.8.10",
    "react": "^0.12.2",
    "reactify": "^0.17.1",
    "vinyl-source-stream": "^1.0.0"
  }
}
```

package.jsonをリポジトリに追加して、GitHubにpushしておきます。

### git cloneし直す

GitHubにpushしたリポジトリを別のディレクトリにcloneして動作確認をします。

``` bash
$ cd ~/react_apps
$ git clone https://github.com/masato/helloworld-gulp.git gulp-test
```

プロジェクトのディレクトリに移動して、`npm install`を実行してローカルにモジュールをインストールします。`gulp browserify`タスクを実行してJSXファイルをコンパイルします。

``` bash
$ cd gulp-test
$ npm install
$ gulp browserify
```

ローカルでSimpleHTTPServerを起動して、ブラウザから動作確認します。

``` bash
$ python -m SimpleHTTPServer 8080
```

