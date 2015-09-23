title: "Framework7でHTML5モバイルアプリをつくる - Part4: Geolocation APIを使って国コードと通貨コードを逆ジオコーディングする"
date: 2015-09-18 07:59:46
tags:
 - HTML5
 - Framework7
 - Geolocation
 - ジオコーディング
 - YQL
 - OpenData
 - GoogleMapsAPI
description: Framework7でモバイルWebを開発するとHTML5/JavaScript/CSSを使ってiOSやMaterialデザインのネイティブに近いUIを作ることができます。またW3Cが策定しているHTML5のデバイスAPIを活用すればネイティブのAPIをコールしなくてもHTML5経由でハードウェアの制御を行うこともできます。簡単なところからGeolocation APIで緯度経度情報を取得して遊んでみようと思います。
---

[Framework7](http://www.idangero.us/framework7/)でモバイルWebを開発するとHTML5/JavaScript/CSSを使ってiOSやMaterialデザインのネイティブに近いUIを作ることができます。またW3Cが策定しているHTML5のデバイスAPIを活用すれば、ネイティブのAPIをコールしなくてもHTML5経由でハードウェアの制御を行うこともできます。簡単なところから[Geolocation API](http://dev.w3.org/geo/api/spec-source.html)で緯度経度情報を取得して遊んでみようと思います。

参考

* [HTML5 compatibility on mobile and tablet browsers with testing on real devices](http://mobilehtml5.org/)
* [Mobile ♥ JavaScript Hardware Access & Device APIs](http://www.girliemac.com/presentation-slides/html5-mobile-approach/deviceAPIs.html)
* [デバイスAPIはどこまで使える？最新事情を紹介](https://codeiq.jp/magazine/2014/12/20086/)

<!-- more -->

## デモアプリ

リポジトリは[こちら](https://github.com/masato/docker-framework7)です。デモは[こちら](https://html5mobile.space/)から実行できます。日本国内だと現在位置の緯度経度から計算できる国コードはJP、通貨コードはJPYだけなのであまりおもしい結果が出ませんが、任意の緯度・経度を手動で入力すれば逆ジオコーディングで位置情報から国コードや通貨コードを取得することができます。

![html5mobile-index.png](/2015/09/18/framework7-geolocation-api/html5mobile-index.png)

### 緯度・経度を入力する

緯度経度をGeolocation APIを使わずにフォームへ手動入力してみます。「位置を入力する」タブからテスト用にアゼルバイジャンの緯度・経度を設定します。

* 緯度: 40.406605
* 経度: 49.804306

![html5mobile-input.png](/2015/09/18/framework7-geolocation-api/html5mobile-input.png)

フォームに入力したデータは[Form Data](http://www.idangero.us/framework7/docs/form-data.html)にあるようにJSON形式で取得することができます。

```html index.html
<form id="my-form" class="list-block">
  <ul>
    <li>
    <div class="item-content">
      <div class="item-media"><i class="icon icon-form-settings"></i></div>
      <div class="item-inner">
        <div class="item-title label">緯度</div>
        <div class="item-input">
          <input name="latitude" type="text" placeholder="緯度を入力">
        </div>
      </div>
    </div>
  </li>
```

「入力値を使う」ボタンをタップすると`app.formToJSON`からJSON形式でフォームの値を取得できます。

```js src/client/js/index.js
$$('.form-to-json').on('click', function(){
    var formData = app.formToJSON('#my-form');
    if (!isNumber(formData.latitude) ||
        !isNumber(formData.longitude)) {
        return app.alert('数値を入力してください。');
    }

    latLng = new google.maps.LatLng(formData.latitude,
                                    formData.longitude);
    geocodeAction(latLng);
});
```


### 緯度・経度の現在位置を取得

W3Cの[Geolocation API](http://dev.w3.org/geo/api/spec-source.html)を使います。[navigator.geolocation.watchPosition](https://developer.mozilla.org/ja/docs/Web/API/Geolocation/watchPosition)の場合はデバイスの位置が変わる度に位置情報を取得しているため、電池の消耗が激しそうです。

画面のボタンを押したときだけ[navigator.geolocation.getCurrentPosition](https://developer.mozilla.org/ja/docs/Web/API/Geolocation/getCurrentPosition)を使い現在の位置情報を取得するようにします。

```js src/client/js/index.js
$$('#location_btn').on('click', function () {
    if (navigator.geolocation) {
        app.showIndicator();
        navigator.geolocation.getCurrentPosition(
            function(position) {
                latLng = new google.maps.LatLng(position.coords.latitude,
                                   position.coords.longitude);
...
```

## 逆ジオコーディング

「現在位置を使う」または「位置を入力する」のタブから取得した緯度・経度情報を使って逆[ジーコーディング](https://ja.wikipedia.org/wiki/%E3%82%B8%E3%82%AA%E3%82%B3%E3%83%BC%E3%83%87%E3%82%A3%E3%83%B3%E3%82%B0)をします。

### 国コードの取得

[Google Maps Javascript API Google Maps Javascript API
](https://developers.google.com/maps/documentation/javascript/)の[逆ジオコーディング](https://developers.google.com/maps/documentation/javascript/examples/geocoding-reverse)を使います。

```js src/client/js/index.js
function getCountryNameCode(latLng, callback) {
    var geocoder = new google.maps.Geocoder();
    var retval = {};
    var location =  {lat: latLng.lat(), lng: latLng.lng()};

    geocoder.geocode({'location': location}, function(results, status) {
        if (status !== google.maps.GeocoderStatus.OK)
            return callback(new Error("can't get goecoder"));
        _.forEach(results[0].address_components, function (comp) {
            if (comp.types[0] == "country") {
                retval['short'] = comp.short_name;
                retval['long'] = comp.long_name;
            }
        });
        if (!retval)
            return callback(new Error("can't get country code"));
        else
            return callback(null, retval);
    });
}
```

`address_component.types[0] == "country"`のときに、`address_component.short_name`に国コードが入っています。

* long_name: 日本
* short_name: JP

参考

* [Getting street,city and country by reverse geocoding using google](http://stackoverflow.com/questions/18173242/getting-street-city-and-country-by-reverse-geocoding-using-google)
* [Parse json from google maps reverse geocode?](http://stackoverflow.com/questions/22185725/parse-json-from-google-maps-reverse-geocode)


### 通貨コードの取得

通貨コードはGoogle Maps APIの逆ジオコーディングでも取得できなかったので、Open Knowledgeの[Frictionless Open Data](http://data.okfn.org)から[Comprehensive country codes: ISO 3166, ITU, ISO 4217 currency codes and many more](http://data.okfn.org/data/core/country-codes)の変換表を使うことにします。データをJSON形式で[ダウンロード](https://github.com/masato/docker-framework7/blob/master/src/server/country-codes.json)します。


日本を例にすると国コードや通貨コードの他にもFIFAやIOCなど様々な日本のコードが定義されています。

```json country-codes.json
"name":"Japan","name_fr":"Japon","ISO3166-1-Alpha-2":"JP","ISO3166-1-Alpha-3":"JPN","ISO3166-1-numeric":"392","ITU":"J","MARC":"ja","WMO":"JP","DS":"J","Dial":"81","FIFA":"JPN","FIPS":"JA","GAUL":"126","IOC":"JPN","currency_alphabetic_code":"JPY","currency_country_name":"JAPAN","currency_minor_unit":"0","currency_name":"Yen","currency_numeric_code":"392","is_independent":"Yes"}
```

* ISO3166-1-Alpha-2: JP
* currency_alphabetic_code: JPY

Expressで簡単なAPIサーバーを用意して、ここからダウンロードした[JSONデータ](https://github.com/masato/docker-framework7/blob/master/src/server/country-codes.json)をクエリするようにしました。

クライアントサイドからはGoecoderから取得したshort_name(JP)を条件にしてサーバーサイドのAPIを実行します。

```js src/client/js/index.js
var q = '/api/currency/' + countryShortName;
app.showIndicator();
$$.get(q, function(data) {
    app.hideIndicator();
    data = JSON.parse(data);
    currencyCode = data.currency_code;
    $$('#currency_code').text(currencyCode);
});
```

サーバーサイドの[app.js](https://github.com/masato/docker-framework7/blob/master/src/server/app.js)では引数のISO3166-1-Alpha-2(JP)からcurrency_alphabetic_code(JPY)を[lodash](https://lodash.com/)の[_.find](https://lodash.com/docs#find)を使ってクエリします。

```js src/server/app.js
"use strict";

var express = require('express'),
    path = require('path'),
    bodyParser = require('body-parser'),
    _ = require('lodash'),
    countryCodes = require('./country-codes.json'),
    router = express.Router(),
    app = express();

app.use(express.static(path.join(__dirname,'..','..','dist')));
app.use(bodyParser.urlencoded({ extended: true }));
app.use(bodyParser.json());

router.get('/currency/:country', function(req, res) {
    var currency_code = _.result(_.find(countryCodes,
                                   'ISO3166-1-Alpha-2', req.params.country),
                                   'currency_alphabetic_code');
    res.json({ currency_code: currency_code});
});

app.use('/api', router);

var server = app.listen(3000, function () {
  var host = server.address().address,
      port = server.address().port;

  console.log('Example app listening at http://%s:%s', host, port);
});
```

### 為替レート

メニューの「為替レート」をタップします。取得した通貨コードを使って[YQLから為替レートを](/2015/09/14/yql-currency/)を取得してみます。この例では[アゼルバイジャン・マナト](https://ja.wikipedia.org/wiki/%E3%82%A2%E3%82%BC%E3%83%AB%E3%83%90%E3%82%A4%E3%82%B8%E3%83%A3%E3%83%B3%E3%83%BB%E3%83%9E%E3%83%8A%E3%83%88)の為替レートです。

![html5mobile-currency.png](/2015/09/18/framework7-geolocation-api/html5mobile-currency.png)

1ドル、1ユーロ、1円の現地通貨のレートを[YQL](https://developer.yahoo.com/yql)の`yahoo.finance.xchange`からクエリします。

```js index.js
function currencyRate() {
    var base_url = 'https://query.yahooapis.com/v1/public/yql';
    var query = 'select%20*%20from%20yahoo.finance.xchange%20where%20pair%20in%20("'+'USD'+currencyCode+','+'EUR'+currencyCode+','+'JPY'+currencyCode+'")';
    var q = base_url + '?q=' + query + '&format=json&env=store%3A%2F%2Fdatatables.org%2Falltableswithkeys';
    app.showIndicator();
    $$.get(q, function(data) {
        app.hideIndicator();
        data = JSON.parse(data);
        if (!data.query || !data.query.results) return;
        var rate = data.query.results.rate;
        console.log(rate);
        rate.forEach(buildRate);
    });
}
```
