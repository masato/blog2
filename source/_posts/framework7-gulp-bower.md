title: "Framework7でHTML5モバイルアプリをつくる - Part2: GulpとBowerでパッケージを1つにまとめる"
date: 2015-09-10 12:03:28
tags:
 - Framework7
 - HTML5モバイルアプリ
 - Bower
 - Gulp
description: 前回はFramework7のパッケージはBowerからインストールした後、HTMLファイルから直接bower_componentsディレクトリからJavaScriptとCSSをロードしていました。Gulpから事前にBowerを使ってインストールしたパッケージを公開用ディレクトリにパブリッシュすることが今回の目的です。今のところFramework7だけしか使っていませんが、bower_components配下にインストールされたパッケージのJavaScriptやCSSを1つのファイルにまとめて、本番用にファイルスサイズを圧縮させます。
---


[前回は](/2015/08/28/framework7-getting-started/)Framework7のパッケージは[Bower](http://bower.io/)からインストールした後、HTMLファイルから直接`bower_components`ディレクトリからJavaScriptとCSSをロードしていました。[Gulp](http://gulpjs.com/)から事前に[Bower](http://bower.io/)を使ってインストールしたパッケージを公開用ディレクトリにパブリッシュすることが今回の目的です。今のところ[Framework7](https://github.com/nolimits4web/Framework7)だけしか使っていませんが、`bower_components`配下にインストールされたパッケージのJavaScriptやCSSを1つのファイルにまとめて、本番用にファイルスサイズを圧縮させます。


<!-- more -->


## プロジェクト


Gulpタスクから公開用ディレクトリにBowerパッケージをコピーした後のディレクトリ構造は以下のようになります。リポジトリは[こちら](https://github.com/masato/docker-framework7/tree/gulp)です。

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
├── gulpfile.js
├── node_modules -> /dist/node_modules
├── package.json
└── public
    ├── about.html
    ├── css
    ├── dist
    │   ├── css
    │   │   ├── vendor.css
    │   │   └── vendor.min.css
    │   └── js
    │       ├── vendor.js
    │       └── vendor.min.js
    ├── index.html
    └── js
        └── index.js
```



## package.json

GulpからBowerパッケージをまとめる方法はいくつかありますが、今回は以下のプラグインを使います。[package.json](https://github.com/masato/docker-framework7/blob/master/package.json)のdevDependenciesフィールドに必要なプラグインを定義します。

```json package.json
{
    "name": "framework7-docker",
    "description": "framework7 docker",
    "version": "0.0.1",
    "private": true,
    "dependencies": {
        "express": "~4.13.3"
    },
    "devDependencies": {
        "bower": "~1.5.2",
        "gulp": "~3.9.0",
        "gulp-util": "~3.0.6",
        "gulp-concat": "~2.6.0",
        "gulp-filter": "~3.0.1",
        "gulp-uglify": "~1.4.0",
        "gulp-minify-css": "~1.2.1",
        "gulp-rename": "~1.2.2",
        "main-bower-files": "~2.9.0"
    },
    "scripts": {
        "start": "node app.js",
        "postinstall": "bower install",
        "bower": "gulp bower"
    }
}
```

## main-bower-files

### dependencies

[main-bower-files](https://github.com/ck86/main-bower-files)はプロジェクトの[bower.json](https://github.com/masato/docker-framework7/blob/master/bower.json)のdependenciesに定義されたパッケージを参照します。今回の例ではFramework7だけです。

```json bower.json
{
    "name": "framework7-docker",
    "version": "0.0.1",
    "private": true,
    "dependencies": {
        "framework7": "~1.2.0"
    },
    "overrides": {
        "framework7": {
            "main": [
                "dist/js/framework7.js",
                "dist/css/framework7.ios.css",
                "dist/css/framework7.ios.colors.css"
            ]
        }
    }
}
```

[main-bower-files](https://github.com/ck86/main-bower-files)をrequireしてコールすると、Bowerパッケージのbower.jsonの`main`セクションに定義されたエントリポイントファイル名を配列で返してくれます。


```js
var bower = require('main-bower-files');
console.log(bower());
```

実行結果

```bash
[ '/app/bower_components/framework7/dist/js/framework7.js',
  '/app/bower_components/framework7/dist/css/framework7.ios.css',
  '/app/bower_components/framework7/dist/css/framework7.ios.colors.css' ]
```


Gulpタスクでは取得したエントリポイントのファイル名からストリームで中身を結合して圧縮していく処理をつないでいきます。

### overrides

ただし、Framework7の[bower.json](https://github.com/nolimits4web/Framework7/blob/master/bower.json)の`main`セクションは以下のようにディレクトリになっているため[main-bower-files](https://github.com/ck86/main-bower-files)はファイル名が取得できません。

```json bower.json
...
  "main": "dist/",
...
```

プロジェクトのbower.jsonに`overrides`セクションを定義して、パッケージ毎の`main`セクションを上書きすることができます。下の例ではJavaScriptとCSSファイルをエントリポイントに追加します。


```json bower.json
...
    "overrides": {
        "framework7": {
            "main": [
                "dist/js/framework7.js",
                "dist/css/framework7.ios.css",
                "dist/css/framework7.ios.colors.css"
            ]
        }
    }
...
```


## gulpfie.js

[gulpfile.js](https://github.com/masato/docker-framework7/blob/master/gulpfile.js)を作成します。bowerJSとbowerCSSタスクはJavaScriptとCSSの違いはありますが処理内容は同じです。

1. main-bower-filesからパッケージのエントリポイントのファイル名を配列で取得する
1. JavaScriptかCSSのどちらかにフィルタする
1. ファイルの中身を1つにまとめて、公開用ディレクトリに出力する
1. 1つにまとめたファイルの中身を圧縮して、公開用ディレクトリに出力する


```js gulpfie.js
"use strict";

var gulp = require('gulp'),
    gutil = require('gulp-util'),
    concat = require('gulp-concat'),
    rename = require('gulp-rename'),
    uglify = require('gulp-uglify'),
    minifyCss = require('gulp-minify-css'),
    gulpFilter = require('gulp-filter'),
    bower = require('main-bower-files');

var publishDir = 'public/dist',
    dist = {
        js: publishDir + '/js/',
        css: publishDir + '/css/'
    };

gulp.task('bowerJS', function() {
    var jsFilter = gulpFilter('**/*.js');
    return gulp.src(bower())
           .pipe(jsFilter)
           .pipe(concat('vendor.js'))
           .pipe(gulp.dest(dist.js))
           .pipe(uglify())
           .pipe(rename({
               extname: '.min.js'
            }))
           .pipe(gulp.dest(dist.js));
});

gulp.task('bowerCSS', function() {
    var cssFilter = gulpFilter('**/*.css');

    return gulp.src(bower())
           .pipe(cssFilter)
           .pipe(concat('vendor.css'))
           .pipe(gulp.dest(dist.css))
           .pipe(minifyCss())
           .pipe(rename({
               extname: '.min.css'
            }))
           .pipe(gulp.dest(dist.css))
});

gulp.task('bower', ['bowerJS', 'bowerCSS']);
gulp.task('default', ['bower']);
```

package.jsonのscriptsセクションに`"bower": "gulp bower"`を定義していて、npn run scriptsとして実行して、bowerJSとbowerCSSタスクをまとめて実行します。

```bash
$ npm run bower
```

Docker Composeを使って実行します。

```bash
$ docker-compose run --rm f7 npm run bower
npm info it worked if it ends with ok
npm info using npm@2.13.3
npm info using node@v3.2.0
npm info prebower framework7-docker@0.0.1
npm info bower framework7-docker@0.0.1

> framework7-docker@0.0.1 bower /app
> gulp bower

[17:25:57] Using gulpfile /app/gulpfile.js
[17:25:57] Starting 'bowerJS'...
[17:25:57] Starting 'bowerCSS'...
[17:26:11] Finished 'bowerJS' after 14 s
[17:26:11] Finished 'bowerCSS' after 14 s
[17:26:11] Starting 'bower'...
[17:26:11] Finished 'bower' after 18 μs
npm info postbower framework7-docker@0.0.1
npm info ok
```

## index.html

最後に`bower_components`を参照していた[index.html](https://github.com/masato/docker-framework7/blob/master/public/index.html)ファイルを編集します。JavaScriptとCSSファイルを公開用の`dist`ディレクトリからロードするように変更します。

```html index.html
...
<head>
...
    <link rel="stylesheet" href="/dist/css/vendor.css">
</head>
...
<body>
...
    <script type="text/javascript" src="/dist/js/vendor.js"></script>
...
</body>
```