title: "Electric Impのimp Moduleとimp Cloud"
date: 2015-03-03 23:13:02
tags:
 - ElectricImp
 - Cylonjs
 - impCloud
 - impModule
 - SparkCore
 - SeeedStudio
description: Cylon.jsのPlatformsページを見ているとElectric Imp用のモジュールが気になりました。Spark Cloudと似たコンセプトですが、Node.jsで書いてimp CloudにデプロイするAgentが特徴的です。
---

[Cylon.js](http://cylonjs.com/)の[Platforms](http://cylonjs.com/documentation/platforms/)ページを見ていると[Electric Imp](https://electricimp.com/)用[モジュール](http://cylonjs.com/documentation/platforms/imp/)が気になりました。[Spark Cloud](https://www.spark.io/features)と似たコンセプトですが、Node.jsで書いて[imp Cloud](https://electricimp.com/product/cloud/)にデプロイするAgentが特徴的です。

<!-- more -->

## imp Module

[imp Module](https://electricimp.com/product/hardware/)はimp001 Card、imp002 Module、imp003、imp004の4種類があります。[データシート](http://electricimp.com/docs/hardware/imp/datasheets/)を読むのがわかりやすいです。imp001 Cardが[開発用](https://electricimp.com/docs/gettingstarted/)モジュールで、imp002以降が本番用のモジュールみたいです。Spark Coreと同様にWi-Fiモジュールを組み込んでいるimp Cloudを通してアプリのデプロイやデバイスの管理ができます。imp Moduleとつなぐことで他のいろいろなデバイスをコネクテッドデバイスにすることができます。SparkFunでは[Android用のシールド](https://www.sparkfun.com/products/12887)も販売しています。

[Buy a Dev Kit](https://electricimp.com/docs/gettingstarted/devkits/)にはリンクがありませんが、セットで安かったので[Seeed Studio](http://www.seeedstudio.com)から購入しました。[Maker Shed](http://www.makershed.com/products/electric-imp#)のセット方が$35.98でちょっと安いです。

## imp Cloud

WebIDEの[imp IDE](https://electricimp.com/product/ide/)を使いAgentとDeviceのコードを2種類書く必要があります。Deviceの方は[Firmata](http://arduino.cc/en/reference/firmata)ファームウェアと違ってNode.jsで書けるので、digitalWriteやanalogWrite などSparkの[Tinker API](http://docs.spark.io/tinker/#tinkering-with-tinker-the-tinker-api)くらいのAPIを用意しておけば、とりあえずLチカから始められそうです。

[imp Cloud](https://electricimp.com/product/cloud/)にAgentをデプロイするとリモートからimpデバイスをコントロールすることができます。Spark Cloudにもpub/subのようなコードをデプロイできますが、imp Cloudのagentの方がより柔軟に書けそうです。クラウド上にエージェントやゲートウェイとしてデバイスを操作するプログラムが自由に書けると、いろいろなことができそうで楽しみです。

## Seeed Studioでお買い物

[Seeed Studio](http://www.seeedstudio.com)は日本からでも50ドル以上の買い物は[送料無料](http://www.seeedstudio.com/depot/shippinginfo.html)になります。[imp cardと imp April Breakout Boardのセット](http://www.makershed.com/products/electric-imp)で$38.00ですが、送料無料に足りないので他にもいくつか購入しました。[Grove Starter Kit](http://www.seeedstudio.com/depot/Electric-Imp-a-WiFi-enabled-Development-Platform-p-1971.html)とか[Espruino](http://www.seeedstudio.com/depot/Espruino-Board-v14-p-2200.html)とか目移りしてしまいます。
