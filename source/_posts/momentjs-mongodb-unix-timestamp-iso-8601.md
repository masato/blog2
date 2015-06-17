title: "Moment.jsを使ってUNIXタイムスタンプをISO 8601表記に変換する"
date: 2015-04-07 22:58:53
tags:
 - Momentjs
 - Nodejs
 - MongoDB
 - ISO8601
 - Docker
description: ISO 8601のUTCタイムゾーンでtimestampが保存されたMongoDBがあります。レンジクエリを書くときにISO 8601で引数を渡していますが、UNIXタイムスタンプを使ってクエリできないか調べました。Moment.jsを使うといろいろな表記の日付をパースしたりフォーマットすることができます。
---

ISO 8601のUTCタイムゾーンでtimestampが保存されたMongoDBがあります。レンジクエリを書くときにISO 8601で引数を渡していますが、UNIXタイムスタンプを使ってクエリできないか調べました。[Moment.js](http://momentjs.com/)を使うといろいろな表記の日付をパースしたりフォーマットすることができます。


## テスト用のDockerコンテナ

Dockerイメージをビルドするプロジェクトを作成します。

``` bash
$ mkdir -p ~/node_apps/moment-spike
$ cd !$
```

```json ~/node_apps/moment-spike/package.json
{
    "name": "moment-spike",
    "version": "0.0.1",
    "private": true,
    "dependencies": {
       "moment": "~2.8.1"
    },
    "scripts": {"start": "node"}
}
```

Dockerfileを作成してイメージをビルドします。ベースイメージは[google/nodejs-runtime](https://registry.hub.docker.com/u/google/nodejs-runtime/)を使います。

``` bash
$ echo FROM google/nodejs-runtime > Dockerfile
$ docker pull google/nodejs-runtime
$ docker build -t moment-spike .
```

## 使い方

使い捨てのDockerコンテナを起動します。

``` bash
$ docker run --rm -it moment-spike 

> moment-spike@0.0.1 start /app
> node

>
```

### moment(timestamp * 1000)

UNIXタイムスタンプの作成は`moment().unix()`を使います。

``` js
> var moment = require('moment')
undefined
> start = moment().unix()
1428393579
```

UNIXタイムスタンプをフォーマットするときは``X``を指定します。単位が秒のまま引数に渡すとパースできません。

``` js
> moment(start).format()
'1970-01-17T12:46:33+00:00'
> moment(start,'X').format();
'2015-04-07T07:59:39+00:00'
```

[Unix Offset (milliseconds)](http://momentjs.com/docs/#/parsing/unix-offset/)にあるように、`moment(Number);`のNumberはJavaScriptの`new Date(Number)`と同様にミリ秒です。`moment(timestamp * 1000)`のように1000倍にしてもパースできます。

``` js
> moment(start*1000).format()
'2015-04-07T07:59:39+00:00'
```

### ISO 8601表記にする

Moment.jsで作成した日付をISO 8601表記にする場合は[Date.prototype.toISOString](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/Date/toISOString)関数を使います。

``` js
> moment(start*1000).toISOString();
'2015-04-07T07:59:39.000Z'
```
