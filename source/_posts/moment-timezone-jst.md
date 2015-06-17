title: "Moment Timezoneを使って日付をJSTに変換する"
date: 2015-04-08 15:12:39
tags:
 - Momentjs
 - moment-timezone
 - Nodejs
 - MongoDB
 - ISO8601
 - Docker
description: Moment TimezoneはIANAのtz databaseを使ってMoment.jsにタイムゾーン操作のAPIを追加してくれます。前回確認したMoment.jsでも+0900するとJSTになりますが自分で計算するのが面倒です。Asia/Tokyoのようにタイムゾーンを指定して変換できると便利です。
---

[Moment Timezone](http://momentjs.com/timezone/)はIANAの[tz database](http://ja.wikipedia.org/wiki/Tz_database)を使って[Moment.js](http://momentjs.com/)にタイムゾーン操作のAPIを追加してくれます。[前回](/2015/04/07/momentjs-mongodb-unix-timestamp-iso-8601/)確認したMoment.jsでも`+09:00`するとJSTになりますが自分で計算するのが面倒です。`Asia/Tokyo`のようにタイムゾーンを指定して変換できると便利です。

<!-- more -->

## テスト用のDockerコンテナ

Dockerイメージをビルドするプロジェクトを作成します。

``` bash
$ mkdir -p ~/node_apps/moment-tz-spike
$ cd !$
```

package.jsonに`moment-timezone`パッケージを指定します。

```json ~/node_apps/moment-tz-spike/package.json
{
    "name": "moment-tz-spike",
    "version": "0.0.1",
    "private": true,
    "dependencies": {
       "moment-timezone": "0.3.1"
    },
    "scripts": {"start": "node"}
}
```

Dockerイメージをビルドしてコンテナを起動します。

```bash
$ docker build -t moment-tz-spike .
$ docker run --rm -it moment-tz-spike

> moment-tz-spike@0.0.1 start /app
> node

>
```

## 使い方

タイムゾーンを`Asia/Tokyo`に指定して現在時刻をISO 8601表示の日付にフォーマットします。

```js
> var moment = require('moment-timezone');
> moment().tz("Asia/Tokyo").format();
'2015-04-08T10:29:25+09:00'
```

UTCからJSTに変換します。

```js
> moment('2015-04-08T02:36:45.408Z').tz("Asia/Tokyo").format()
'2015-04-08T11:36:45+09:00'
```

指定したISO 8601表示の日付をUNIXタイムスタンプに変換します。

```js
> moment('2015-04-08T02:36:45.408Z').tz('Asia/Tokyo').unix()
1428460605
```

UNIXタイムスタンプからISO 8601表示の日付にフォーマットします。

```js
> moment(1428460605,'X').tz("Asia/Tokyo").format()
'2015-04-08T11:36:45+09:00'
```
