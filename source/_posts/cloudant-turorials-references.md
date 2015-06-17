title: "Cloudantのチュートリアル - Part1: はじめに"
date: 2015-01-08 01:45:23
tags:
 - Cloudant
 - Bluemix
 - Nodejs
 - MBaaS
description: Cloudantが毎月$50ドルまで利用料が無料になるそうです。こちらに記事があります。Build More with $50 Free Each Month $50でどれだけ使えるかというと、$50 = 1.6 million+ light API requests (reads) $50 = 300,000+ heavy API requests (writes) $50 = 50 GB of storage Pricingのページにも$50未満は請求されないと書いてあります。No charge *ever* if your monthly usage is under $50.00 USD BluemixとNode.jsのサンプルが多いですが、developerWorksを中心にチュートリアルをあつめてみました。MBaaSとしてどんな使い方ができるか勉強していこうと思います。
---

Cloudantが毎月$50ドルまで利用料が無料になるそうです。毎月$50ドルまでの利用料が無料になるそうです。こちらに記事があります。[Build More with $50 Free Each Month](https://cloudant.com/blog/build-more-with-50-free-each-month/)

$50でどれだけ使えるかというと、

* $50 = 1.6 million+ light API requests (reads)
* $50 = 300,000+ heavy API requests (writes)
* $50 = 50 GB of storage

[Pricing](https://cloudant.com/product/pricing/)のページにも$50未満は請求されないと書いてあります。
> No charge *ever* if your monthly usage is under $50.00 USD

MBaaSとしてどんな使い方ができるか勉強していこうと思います。

<!-- more -->


### Getting Started

Getting StartedやAPIリファレンス、サンプルアプリなどCloudantを学習するにはまずここから始めます。

* [For Developers](https://cloudant.com/for-developers/)
* [Cloudant API Reference](https://docs.cloudant.com/)


### チュートリアル

BluemixとNode.jsのサンプルが多いです。developerWorksを中心にチュートリアルをあつめてみました。Javaの開発のときはdeveloperWorksをよく読んでいました。相変わらずクオリティが高くありがたいです。

* [Bluemix 上で稼動する Web アプリケーション開発方法 - Node.js 編](http://www.ibm.com/developerworks/jp/cloud/library/j_cl-bluemix-nodejs-app/)
* [Bluemix 上で Cloudant を使用してシンプルなワード・ゲームを作成する](http://www.ibm.com/developerworks/jp/cloud/library/cl-guesstheword-app/)
* [Cloudant と Bluemix を利用した動的な Google Gauge](http://www.ibm.com/developerworks/jp/cloud/library/cl-dynamicgooglegauge-app/)
* [Cloudant を使用して Bluemix 上で Famo.us モバイル・アプリを自動化する](http://www.ibm.com/developerworks/jp/cloud/library/cl-bluemix-famous-mobile/)
* [エンタープライズ クラウド ストレージ入門： Cloudant データベースの構築方法](https://jp.linux.com/Linux%20Jp/tutorial/424880-tutorial2014122701)
* [エンタープライズ クラウド ストレージ入門その 2： Cloudant データベースへのアクセス方法](https://jp.linux.com/news/linuxcom-exclusive/424913-lco2014122901)

### Example Applications

インタラクティブなデモになっているので実際に操作することができます。

* [Cloudant examples in different programming languages](https://github.com/cloudant/haengematte)
* [View Aggregation](http://examples.cloudant.com/simplegeo_places/_design/geo/index.html)
* [Chained MapReduce](http://examples.cloudant.com/sales/_design/sales/index.html)
* [Full Text Indexing and Query](http://examples.cloudant.com/lobby-search/_design/lookup/index.html)
* [Geo-spatial queries](http://examples.cloudant.com/simplegeo_places/_design/geo/search.html)
* [Data Visualization](http://examples.cloudant.com/whatwouldbiebersay/_design/whatwouldbiebersay/index.html)