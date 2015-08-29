title: "IoTのためのHTML5モバイルアプリ - Part1: リソース"
date: 2015-08-27 15:33:26
categories:
 - IoT
tags:
 - myThings
 - HTML5
 - IoT
 - Framework7
 - konashi
description: konashi.jsはWebViewを使いインターネットで公開しているHTML5アプリを読み込みます。iOSで開発しなくてもjsdo.it上でHTML5 JavaScript CSSのコードを書けば簡単に実行することができます。スマホのWebブラウザから利用できるHTML5モバイルアプリにすると、より汎用的なIoTクライアントが開発できそうです。
---

[konashi.js](http://konashi.ux-xu.com/kjs/)はWebViewを使いインターネットで公開しているHTML5アプリを読み込みます。iOSで開発しなくても[jsdo.it](http://jsdo.it/)上でHTML5/JavaScript/CSSのコードを書けば簡単に実行することができます。スマホのWebブラウザから利用できるHTML5モバイルアプリにすると、より汎用的なIoTクライアントが開発できそうです。

<!-- more -->

## HTML5ハイブリッドモバイルアプリ

HTML5ハイブリッドモバイルアプリ用のフレームワークには[Cordova](https://cordova.apache.org/)/[PhoneGap](http://phonegap.com/)や[Appcelerator](http://www.appcelerator.com/)などがあります。HTML5/Javascript/CSSを使って、スマホのネイティブアプリを開発することができます。作成したアプリはApp StoreやGoogle Playから配信します。

基本的にはCordova開発のUIコンポーネントですが、デスクトップブラウザのUIとしても使えます。

* [Ionic](http://ionicframework.com/)
* [OnsenUI](http://ja.onsen.io/)

## HTML5モバイルアプリ

スマホのWebブラウザからアクセスして使うWebアプリを開発するために使います。レスポンシブに対応しているので、モバイルファーストでデザインしてデスクトップブラウザからも利用できます。

[Framework7](http://www.idangero.us/framework7/)はiOSとAndroidのネイティブアプリに近いUIを開発できます。[Sencha Touch](https://www.sencha.com/products/touch/)はエンタープライズアプリの[ExtJS](https://www.sencha.com/products/extjs/)に慣れていれば使いやすいです。

* [Ratchet](http://goratchet.com/)
* [Sencha Touch](https://www.sencha.com/products/touch/)
* [Framework7](http://www.idangero.us/framework7/)
* [Titon](http://titon.io/en)

### HTML5のデバイス系API

W3Cで策定されているデバイス系APIを使うと、スマホに搭載されているセンサー類をHTML5/JavaScriptから制御することができます。特に[Geolocation API](http://www.w3.org/TR/geolocation-API/)や[Vibration API](http://www.w3.org/TR/vibration/)などは、簡単なフィジカル・コンピューティングのアプリ開発に向いていると思います。konashi.jsからも利用できます。

* [デバイスAPIはどこまで使える？最新事情を紹介](https://codeiq.jp/magazine/2014/12/20086/)に
* [【保存版】スマートフォンって実はセンサーの塊！](http://ics-web.jp/lab/archives/4095)
* [エンタープライズモバイルアプリの本命技術、「HTML5ハイブリッドアプリ」とは](http://codezine.jp/article/detail/8141)

### 課題

一番大きな課題はJavaScriptのコードがインターネットから見えてしまうので、連係したいWebサービスのAPIキーなどの情報を安全に保存できないということです。クラウド上でサーバーサイドを用意したり。BaaSを使って安全にセキュリティに関するデータを保存する必要があります。または[myThings](http://mythings.yahoo.co.jp/)のようなアプリを使って、OAuthのマッシュアップを代行してもらうのもよいと思います。
