title: "Framework7でHTML5モバイルアプリをつくる - Part3: gulp-webserverとgulp-watchでLiveReloadする"
date: 2015-09-11 10:22:46
tags:
 - Framework7
 - HTML5モバイルアプリ
 - Bower
 - Gulp
description: Gulpプラグインを使い開発用サーバ-の起動とライブリロードの設定をします。開発用サーバーはExpressサーバーとは独立して起動するのでフロントサイドのSPAの開発が行うことができます。リポジトリはこちらです。
---

Gulpプラグインを使い開発用サーバ-の起動とライブリロードの設定をします。開発用サーバーはExpressサーバーとは独立して起動するのでフロントサイドのSPAの開発が行うことができます。リポジトリは[こちら](https://github.com/masato/docker-framework7/tree/watch)です。

<!-- more -->

## パッケージの追加

[package.json](https://github.com/masato/docker-framework7/blob/watch/package.json)のdevDependenciesに今回の作業で追加するパッケージを定義します。

* [gulp-watch](https://www.npmjs.com/package/gulp-watch)
* [gulp-webserver](https://www.npmjs.com/package/gulp-webserver)
* [del](https://www.npmjs.com/package/del)
* [run-sequence](https://www.npmjs.com/package/run-sequence)

```json package.json
...
    "devDependencies": {
...
        "gulp-watch": "~4.3.5",
        "gulp-webserver": "~0.9.1",
        "del": "~2.0.2",
        "run-sequence": "~1.1.2"
    },
    "scripts": {
        "start": "node src/server/app.js",
        "postinstall": "bower install",
        "clean": "gulp clean",
        "bower": "gulp bower",
        "build": "gulp build",
        "dev-server": "gulp"
    }
}
```

### インストール

Dockerイメージのビルド課程で`RUN npm install`が実行されます。

```bash Dockerfile
...
COPY package.json /app/
COPY bower.json /app/

RUN  npm install
```

package.jsonのscriptsセクションに定義してあるpost install`セクションからBowerパッケージもDockerイメージにインストールされます。

```bash
$ docker-compose run --rm f7 npm run build
```

## config.js

Gulpのビルドタスクでディレクトリ名やファイル名はまとめて[config.js](https://github.com/masato/docker-framework7/blob/watch/config.js)に定義しておきます。相対パスの辺りの記述は現在ではふつうに連結して使えました。

```js config.js
'use strict';
var path = require('path');

var filter = {
        html: '**/*.html',
        js: '**/*.js',
        css: '**/*.css',
    },
    srcDir = './src',
    /*relativeSrcPath = path.relative('.', srcDir),*/
    relativeSrcPath = srcDir,
    src = {
        server: relativeSrcPath + '/server/',
        client: relativeSrcPath + '/client/',
        scripts: relativeSrcPath + '/client/js/' + filter.js,
        views: './views/' + filter.html
    },
    distDir = './dist',
    dist = {
        base: distDir,
        js: distDir + '/js/',
        css: distDir + '/css/'
    },
    output = {
        buildJs: 'bundle.js',
        buildCss: 'bundle.css',
        bowerJs: 'vendor.js',
        bowerCss: 'vendor.css',
        minJs: '.min.js',
        minCss: '.min.css'
    };

module.exports = {
    src: src,
    dist: dist,
    filter: filter,
    output: output
}
```

## Gulpビルドタスク (del, run-sequence)

以下のサイトを参考にビルドタスクを書いていきます。

* [Gulp でタスクを並列/直列処理する](http://qiita.com/naoiwata/items/4c82140a5fb5d7bdb3f8)


### gulpfile.js

最初に作成した[gulpfile.js](https://github.com/masato/docker-framework7/blob/watch/gulpfile.js)の全文です。

```js gulpfile.js
'use strict';

var gulp = require('gulp'),
    gutil = require('gulp-util'),
    concat = require('gulp-concat'),
    rename = require('gulp-rename'),
    uglify = require('gulp-uglify'),
    minifyCss = require('gulp-minify-css'),
    gulpFilter = require('gulp-filter'),
    bower = require('main-bower-files'),
    webserver = require('gulp-webserver'),
    watch = require('gulp-watch'),
    del = require('del'),
    runSequence = require('run-sequence'),
    config = require('./config');

gulp.task('clean', function(cb) {
    return del(config.dist.base, cb);
});

gulp.task('copy', function() {
    return gulp.src(config.src.views)
               .pipe(gulp.dest(config.dist.base));
});

gulp.task('scripts', function() {
    return gulp.src(config.src.scripts)
        .pipe(concat(config.output.buildJs))
        .pipe(gulp.dest(config.dist.js))
        .pipe(uglify())
        .pipe(rename({
            extname: config.output.minJs
        }))
        .pipe(gulp.dest(config.dist.js));
});

gulp.task('bowerJs', function() {
    var jsFilter = gulpFilter(config.filter.js);
    return gulp.src(bower())
           .pipe(jsFilter)
           .pipe(concat(config.output.bowerJs))
           .pipe(gulp.dest(config.dist.js))
           .pipe(uglify())
           .pipe(rename({
               extname: config.output.minJs
            }))
           .pipe(gulp.dest(config.dist.js));
});

gulp.task('bowerCss', function() {
    var cssFilter = gulpFilter(config.filter.css);
    return gulp.src(bower())
           .pipe(cssFilter)
           .pipe(concat(config.output.bowerCss))
           .pipe(gulp.dest(config.dist.css))
           .pipe(minifyCss())
           .pipe(rename({
               extname: config.output.minCss
            }))
           .pipe(gulp.dest(config.dist.css));
});

gulp.task('watch', function () {
    watch(config.src.scripts, function () {
        gulp.start(['scripts']);
    });

    watch(config.src.views, function () {
        gulp.start(['copy']);
    });
});

gulp.task('webserver', function() {
    console.log(config.dist.base);
    return gulp.src(config.dist.base)
        .pipe(webserver({
            host: process.env.DEV_HOST || 'localhost',
            port: process.env.DEV_PORT || 8080,
            livereload: true
        }));
});

gulp.task('bower', ['bowerJs', 'bowerCss']);

gulp.task('build', function(cb) {
  runSequence('clean', ['bower', 'scripts', 'copy'], cb);
});

gulp.task('default', ['webserver','watch']);
```


### buildタスク

buildタスクでは[run-sequence](https://www.npmjs.com/package/run-sequence)パッケージを使いタスクの直列処理と並列処理を制御します。`clean`タスクが終了後に配列で定義したタスクを並列で実行します。

```js gulpfile.js
gulp.task('build', function(cb) {
  runSequence('clean', ['bower', 'scripts', 'copy'], cb);
});
```

## 開発用サーバーとライブリロード (gulp-watch, gulp-webserver)

以下のサイトを参考に開発サーバーのタスクを書いていきます。

* [gulp.js+webpack+stylusでフロントエンド開発環境をつくる](http://blog.flup.jp/2015/02/17/gulp_webpack_stylus/)


### watchタスク

[gulp-watch](https://www.npmjs.com/package/gulp-watch)を使った`watch`タスクを用意して開発しているJavaScriptとHTMLファイルのディレクトリの更新を監視します。JavaScriptとHTMLファイルに編集があった場合にそれぞれ`scripts`タスクと`copy`タスクが実行されます。公開用の`dist`ディレクトリに変換されたファイルがコピーされます。

```js gulpfile.js
gulp.task('watch', function () {
    watch(config.src.scripts, function () {
        gulp.start(['scripts']);
    });

    watch(config.src.views, function () {
        gulp.start(['copy']);
    });
});
```

### webserverタスク

[gulp-webserver](https://www.npmjs.com/package/gulp-webserver)を使い開発用のWebサーバーを起動するタスクです。

```js gulpfile.js
gulp.task('webserver', function() {
    console.log(config.dist.base);
    return gulp.src(config.dist.base)
        .pipe(webserver({
            host: process.env.DEV_HOST || 'localhost',
            port: process.env.DEV_PORT || 8080,
            livereload: true
        }));
});
```

開発をクラウド上で行っているので、 docker-compose.ymlにバインドするホスト名を`environment`セクションの環境変数として`0.0.0.0`にバインドします。また`ports`セクションにDockerホストにマップして公開するポートを定義します。

```yaml docker-compose.yml
  environment:
    - DEV_HOST=0.0.0.0
    - DEV_PORT=8080
  ports:
    - 3030:3000
    - 8090:8080
    - 35729:35729
```

### buildタスク

[run-sequence](https://www.npmjs.com/package/run-sequence)プラグインを使いタスク処理の順番を直列と並列に制御します。

```js gulpfile.js
gulp.task('build', function(cb) {
  runSequence('clean', ['bower', 'scripts', 'copy'], cb);
});

gulp.task('default', ['webserver','watch']);
```

### cleanタスク

[del](https://www.npmjs.com/package/del)パッケージは、プロジェクトをビルドする前の`clean`タスクで公開用ディレクトリをクリアする目的で使います。`return`しないとタスクが途中で止まってしまうので注意が必要です。

```js gulpfile.js
gulp.task('clean', function(cb) {
    return del(config.dist.base, cb);
});
```

## ビルドと実行

同様にbuildコマンドと、

[Run arbitrary package scripts](https://docs.npmjs.com/cli/run-script)にrun-scriptの使い方が書いてあります。package.jsonの`scripts`セクションに定義した`dev-server`コマンドを`npm run dev-server`から実行します。


```bash
$ docker-compose run --rm f7 npm run build
$ docker-compose run --rm --service-ports f7 npm run dev-server
npm info it worked if it ends with ok
npm info using npm@2.13.3
npm info using node@v3.2.0
npm info predev-server framework7-docker@0.0.1
npm info dev-server framework7-docker@0.0.1

> framework7-docker@0.0.1 dev-server /app
> gulp

[21:02:03] Using gulpfile /app/gulpfile.js
[21:02:03] Starting 'webserver'...
./dist
[21:02:03] Webserver started at http://0.0.0.0:8080
[21:02:03] Starting 'watch'...
[21:02:03] Finished 'watch' after 12 ms
[21:02:03] Finished 'webserver' after 62 ms
[21:02:03] Starting 'default'...
[21:02:03] Finished 'default' after 11 μs
```

開発用サーバーは`src/client/js/`と`views`ディレクトリを監視します。ファイルに変更があると、公開用の`dist`ディレクトリにファイルの結合と圧縮を行いコピーします。
