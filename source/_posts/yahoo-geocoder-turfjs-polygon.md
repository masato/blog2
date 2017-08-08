title: "Yahoo!ジオコーダAPIとTurf.jsで住所からポリゴンを計算してgeojson.ioに描画する"
date: 2017-08-04 07:26:07
tags:
 - geocode
 - YOLP
 - GIS
description: Yahoo!ジオコーダAPIで住所文字列から座標情報を取得し、Turf.jsを使いポリゴンを計算します。最後にGeoJSONにフォーマットしてgeojson.ioに描画してみます。
---

　[Yahoo!ジオコーダAPI](https://developer.yahoo.co.jp/webapi/map/openlocalplatform/v1/geocoder.html)で住所文字列から座標情報を取得し、[Turf.js](http://turfjs.org/)を使いポリゴンを計算します。最後にGeoJSONにフォーマットして[geojson.io](http://geojson.io/)に描画してみます。

<!-- more -->

## GIS(地理情報システム)用語

* [GeoJSON](http://geojson.org/)

　地理空間データの標準フォーマットです。[Google Maps](https://www.google.co.jp/maps)や[Mapbox](https://www.mapbox.com/)、[MapQuest](https://www.mapquest.com/)など多くのWeb地図サービスで利用できます。

* 座標情報 (Coordinates)
経度 (Longitude)、緯度 (Latitude)の順に並びます。

　以下は[株式会社パスコ](http://www.pasco.co.jp/)による[用語集](http://www.pasco.co.jp/recommend/word/word022/)を参照します。

* ポイント (Point)
>長さや幅のない対象物を指します。
地図表示の例としては、信号、山頂点、気象観測点、などがあげられます。


* ポリゴン (Polygon)
>境界線を表わす線の終点を始点に一致させ、閉領域を作った面など、地図上で一つの地域を表す多辺図形を一般的にポリゴンと呼びます。 
地図表示の例としては、運動場などがあげられます。

　Mapboxではポイントとポリゴンはこのように表現されます。

![circle.png](/2017/08/04/yahoo-geocoder-turfjs-polygon/circle.png)

## Yahoo!ジオコーダAPI

　[Yahoo!ジオコーダAPI](https://developer.yahoo.co.jp/webapi/map/openlocalplatform/v1/geocoder.html)はYahoo!JAPANが開発している[YOLP](https://developer.yahoo.co.jp/webapi/map/)の地図・地域情報APIの一つです。住所文字列から座標情報(経度、緯度)を取得できます。特に明記はないのですが住所文字列は正規化もしてくれ、`1丁目8番1号`や`１－８－１`、`１丁目8-1`などの揺らぎにもしっかり対応しているようです。

　最初にYahoo! JAPANのWebサービスを利用するために必要なアプリケーションIDを「[アプリケーションIDを登録する](https://www.yahoo-help.jp/app/answers/detail/p/537/a_id/43398)」を参考にして取得しておきます。

## Turf.js

　[Turf.js](http://turfjs.org/)は地理空間分析用のJavaScriptライブラリです。たくさんの地理空間分析[API](http://turfjs.org/docs/)が用意されています。[circle](http://turfjs.org/docs/#circle)メソッドを使い座標情報からポリゴンを計算します。


## geojson.io

　[geojson.io](http://geojson.io/)はブラウザ上で簡単にGeoJSONを編集して地図に描画できるWebサービスです。Mapboxがオープンソースで[開発](https://github.com/mapbox/geojson.io)しています。Node.jsのコードからGeoJSONを出力し[geojsonio-cli](https://github.com/mapbox/geojsonio-cli)を通して使います。

## サンプルプロジェクト

　package.jsonを用意して`npm install`します。


``` json package.json
{
  "name": "ygeocode-sample",
  "description": "ygeocode-sample",
  "version": "0.0.1",
  "private": true,
  "scripts": {
    "start": "node index.js | geojsonio"
  },
  "dependencies": {
    "@turf/turf": "^4.6.1",
    "geojsonio-cli": "^0.2.3",
    "request": "^2.81.0"
  }
}
```

　簡単なNode.jsのスクリプトを書きます。変数は環境に応じて変更してください。

* appId
Yahoo! JAPANのWebサービスのアプリケーションID

* query
ポリゴンを計算する住所文字列、サンプルはヤフオクドームの住所

　以下の[circle](http://turfjs.org/docs/#circle)メソッドの引数はドキュメントに詳細な説明があります。

* radius
ポリゴンの半径

* steps
ポリゴンを描画するステップ数

* units
半径の単位

``` js index.js
'use strict';

const request = require('request');
const turf = require('@turf/turf');

const baseUrl = 'https://map.yahooapis.jp/geocode/V1/geoCoder';
const appId = '<YOUR APP ID>';

const query = '福岡市中央区地行浜2丁目2番2号';
const radius = 1;
const steps = 60;
const units = 'kilometers';

const qs = { appid:appId,
             query:query,
             output:'json'};

request.get({ url:baseUrl, qs:qs, json:true }, (err, response, body) => {
  const features = body.Feature;

  if (features.length < 1) return;

  const feature = features[0];
  const feature = features[0];
  const address = feature.Property.Address;
  const coord = feature.Geometry.Coordinates.split(',');
  const center = { type:'Feature',
                   properties: {},
                   geometry: {
                     coordinates: coord.map((v) => parseFloat(v)),
                     type: 'Point'
                   }
                 };

  const properties = { address:address };
  const polygon = turf.circle(center, radius, steps, units, properties);
  const geojson = {
    type: 'FeatureCollection',
    features: [center, polygon]
  };

  console.log(JSON.stringify(geojson));
});
```

　プログラムを実行するとFeatureCollectionタイプのGeoJSONが出力されます。

{"type":"FeatureCollection","features":[{"type":"Feature","properties":{},"geometry":{"coordinates":[130.36213488,33.59540143],"type":"Point"}},{"type":"Feature","properties":{"address":"福岡県福岡市中央区地行浜2丁目2-2"},"geometry":{"type":"Polygon","coordinates":[[[130.36213488,33.60439182377266],[130.36326319720803,33.60434256833447],[130.36437914850492,33.60419534184053],[130.36547050363063,33.60395175783102],[130.36652530213368,33.60361448586572],[130.3675319845582,33.603187222240585],[130.3684795192171,33.602674649443514],[130.36935752315605,33.60208238479515],[130.37015637598037,33.60141691884032],[130.3708673252925,33.600685544167646],[130.37148258258514,33.59989627544021],[130.3719954085387,33.59905776151671],[130.37240018679,33.598179190628684],[130.37269248536725,33.5972701896556],[130.37286910512242,33.596340718603905],[130.3729281146359,33.59540096144804],[130.37286887121738,33.59446121453118],[130.37269202777998,33.593531773748936],[130.3723995255192,33.59262282175287],[130.3719945724851,33.59174431640888],[130.3714816082883,33.59090588173187],[130.37086625533385,33.59011670248989],[130.3701552571223,33.5893854236307],[130.36935640429795,33.58872005562991],[130.36847844925842,33.58812788679505],[130.3675310102614,33.58761540348344],[130.36652446608005,33.58718821910479],[130.36546984235983,33.586851012683525],[130.36437869091762,33.58660747765104],[130.363262963303,33.58646028142657],[130.36213488,33.58641103622736],[130.36100679669704,33.58646028142657],[130.3598910690824,33.58660747765104],[130.3587999176402,33.586851012683525],[130.35774529391998,33.58718821910479],[130.35673874973867,33.58761540348344],[130.3557913107416,33.58812788679505],[130.35491335570208,33.58872005562991],[130.35411450287774,33.5893854236307],[130.35340350466618,33.59011670248989],[130.35278815171174,33.59090588173187],[130.35227518751492,33.59174431640888],[130.35187023448083,33.59262282175287],[130.35157773222005,33.593531773748936],[130.35140088878265,33.59446121453118],[130.35134164536413,33.59540096144804],[130.3514006548776,33.596340718603905],[130.35157727463277,33.5972701896556],[130.35186957321002,33.598179190628684],[130.35227435146135,33.59905776151671],[130.35278717741488,33.59989627544021],[130.35340243470753,33.600685544167646],[130.35411338401966,33.60141691884032],[130.35491223684397,33.60208238479515],[130.35579024078297,33.602674649443514],[130.35673777544181,33.603187222240585],[130.35774445786635,33.60361448586572],[130.3587992563694,33.60395175783102],[130.3598906114951,33.60419534184053],[130.361006562792,33.60434256833447],[130.36213488,33.60439182377266]]]}}]}
```

　GeoJSONを[geojson.io](http://geojson.io/)のエディタに貼り付けるか、[geojsonio-cli](https://github.com/mapbox/geojsonio-cli)にパイプするブラウザでgeojson.ioのページが直接開きます。

```bash
$ npm start
```

　ヤフオクドームのポイントを中心に半径500メートルのポリンゴンが描画されました。

![polygon.png](/2017/08/04/yahoo-geocoder-turfjs-polygon/polygon.png)
