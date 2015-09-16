title: "YQLを使って為替レートを取得する"
date: 2015-09-14 23:11:11
tags:
 - YQL
 - Yahoo
---

意外に日本では知られていないのですが、[Yahoo! Inc.](https://www.yahoo.com/)の便利な[YQL](https://developer.yahoo.com/yql)を使うと、WebサイトのスクレイピングやWebサービスをSQL風なシンタックスでクエリすることができます。

<!-- more -->

## YQL

[YQL](https://developer.yahoo.com/yql)はYahoo! Query Languageの略語になります。YQLを使うとWebを通してデータのクエリやフィルター、結合などを1つのインタフェースから行うことができます。SQL風なシンタックスなので、開発者に馴染みがあり、データを正確に抽出したい場合に適しています。

## 為替レートを取得する

YQLから[Yahoo! Finance](http://finance.yahoo.com/)の[yahoo.finance.xchange](https://github.com/yql/yql-tables/blob/master/yahoo/finance/yahoo.finance.xchange.xml)テーブルを使います。

以下のURLをHTTP GETするとJSON形式で為替レートを取得することができます。URIエンコードしています。

https://query.yahooapis.com/v1/public/yql?q=select%20*%20from%20yahoo.finance.xchange%20where%20pair%20in%20(%22USDJPY,EURJPY,JPYJPY%22)&format=json&env=store%3A%2F%2Fdatatables.org%2Falltableswithkeys

この例では、USD/JPY、EUR/JPY、JPY/JPYの為替レートを取得しています。それぞれ1ドル、1ユーロ、1円が何円になるかのデータになります。1円は当然ながら1円です。


取得したJSON

```json
{"query":{"count":3,"created":"2015-09-16T14:10:33Z","lang":"ja","results":{"rate":[{"id":"USDJPY","Name":"USD/JPY","Rate":"120.4465","Date":"9/16/2015","Time":"3:10pm","Ask":"120.4490","Bid":"120.4465"},{"id":"EURJPY","Name":"EUR/JPY","Rate":"136.0700","Date":"9/16/2015","Time":"3:10pm","Ask":"136.0800","Bid":"136.0600"},{"id":"JPYJPY","Name":"JPY/JPY","Rate":"1.0000","Date":"9/16/2015","Time":"3:10pm","Ask":"1.0000","Bid":"1.0000"}]}}}
```
