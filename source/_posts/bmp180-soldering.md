title: "気圧と温度センサーのBMP180をはんだ付けする"
date: 2015-03-12 22:08:41
tags:
 - はんだ付け
 - BMP180
 - 電子工作
 - Grove
 - センサー
 - I2C
description: 気圧と温度センサーのBMP180を購入しました。SensorTagやGroveのようにお手軽ではないので、ピンヘッダをはんだ付けする必要があります。村田製作所エレきっず学園のはんだ付けページを読んで基本を勉強します。
---

気圧と温度センサーの[BMP180](https://www.switch-science.com/catalog/1598/)を購入しました。[SensorTag](http://www.tij.co.jp/tool/jp/cc2541dk-sensor)や[Grove System](http://www.seeedstudio.com/wiki/index.php?title=GROVE_System)のようにお手軽ではないので、ピンヘッダをはんだ付けする必要があります。村田製作所[エレきっず学園](http://www.murata.co.jp/elekids/index.html)の[はんだ付け](http://www.murata.co.jp/elekids/ele/craft/knack/soldering/index.html)ページを読んで基本を勉強します。

<!-- more -->

## BMP180

[BMP180](https://www.switch-science.com/catalog/1598/)は[BMP085](https://www.switch-science.com/catalog/1070/)の後継です。[Grove](http://www.seeedstudio.com/wiki/index.php?title=GROVE_System)や[Xadow](http://www.seeedstudio.com/wiki/Xadow)のモジュールにも採用されています。またmbedやArduinoでセンシングするチュートリアルがAdafruitやSparkFunなどのlearnサイトにたくさんあるので勉強できます。

## 電子工作の準備

はんだ付けは工具も手元にないので一式購入することにします。

* [goot 電子工作用はんだこてセット X-2000E](http://www.amazon.co.jp/dp/product/B001PR1KJM)

センサーにはんだ付けするピンヘッダも必要です。

* [普通のピンヘッダ10本セット](https://www.switch-science.com/catalog/92/)

ピンヘッダは1列40ピンなので、必要なピン数にカットします。

* [クラフトツールシリーズ No.40 モデラーズナイフ 74040](http://www.amazon.co.jp/dp/product/B002LE7LB4)

## ブレッドボードに挿して作業する

モジュールキットにピンヘッダを通してブレッドボードに挿してはんだ付けします。
