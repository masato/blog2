title: Spark CoreとSpark CloudからIoTデバイスとクラウドの関係を考える
date: 2015-02-28 21:39:03
tags:
 - SparkCore
 - SparkCloud
 - BaaS
 - IFTTT
description: 昨年購入したまま放置していたSpark Coreをようやく遊べる時間が作れました。Spark CoreはArduino互換のマイコンボードです。Wi-Fiモジュールを搭載しているのでインターネット経由でリモートデバイスのプログラムの更新ができます。SparkデバイスにはSpark CloudというBaaSとセットで使う魅力があります。
---

昨年購入したまま放置していた[Spark Core](https://www.spark.io/)をようやく遊べる時間が作れました。Spark CoreはArduino互換のマイコンボードです。Wi-Fiモジュールを搭載しているのでインターネット経由でリモートデバイスのプログラムの更新ができます。Sparkデバイスには[Spark Cloud](https://www.spark.io/features)というBaaSとセットで使う魅力があります。

<!-- more -->

## IoTデバイスは小型Linuxサーバーかマイコンか

### 小型Linxサーバーとクラウド

一番よく使っている[BeagleBone Black](http://beagleboard.org/)はARM Linuxサーバーとしての楽しみがあります。BeagleBone Blackを搭載した[Ninja Sphere](https://ninjablocks.com/)は[Snappy Ubuntu Core](http://www.ubuntu.com/things)でアプリの管理が可能になり汎用的なデバイスが作れそうです。Intel EdisonはSDカードサイズでWi-FiとBluetoothが使える超小型コンピューターです。Arduinoと組み合わせてGPIOを使うというより、超小型のx86 Linuxサーバーの方に魅力を感じてしまいます。

ただし、どうしてもマシンスペックは低く制約も多いため、クラウドで使えるLinuxサーバーと全く同じには使えません。エッジのLinuxベースのIoTデバイスでちょっと重い処理ができるメリットはありますが、今では500円/月でクラウドのLinuxサーバーが使える時代になりました。コネクテッドデバイスはなるべくハードウェアの制御に特化して、分散処理やデータストアを考えると、なるべくロジックはクラウド上で実装しておいた方がよい気がします。

### マイコンとクラウド

Spark デバイスは[HacksterのSparkプロジェクト](http://spark.hackster.io/)にあるように、ミニマルにハードウェアの制御に向いています。Kickstarterで資金調達している[Spark Electron](https://www.kickstarter.com/projects/sparkdevices/spark-electron-cellular-dev-kit-with-a-simple-data)はさらにおもしろくてSIMカードが刺せます。Spark Cloudやリモートデバイス管理の特徴とあわせると可能性がもっと広がります。

## Spark Cloud

Spark CoreなどのSparkデバイスの魅力はSpark Cloudにつながっているところです。デバイスはSpark Cloudとセキュアな暗号化通信をします。また最初からデバイスがCloudにつながっているので、IFTTTなどのWebサービス連携サービスや自動化と相性が良いです。

### 特徴

[Features](https://www.spark.io/features)に書いてある特徴を簡単にまとめます。

* Arduinoと同じ[Wiring](http://wiring.org.co/)言語や、Node.js、Pythonでもプログラムできる
* ハードウェア制御の[REST API](http://docs.spark.io/api/)と[Spark-CLI](https://github.com/spark/spark-cli)が使える
* インターネット経由でプログラムの書き換えができる
* AtomベースのWeb IDEでコーディングとデプロイができる
* コミュニティーが開発したライブラリをWeb IDEから読み込める
* デバイス間でpub/subのリアルタイム通信
* RSA, AES, SSL/TLSのセキュア通信
* [IFTTT](https://ifttt.com/)の[Spark Channel](http://docs.spark.io/ifttt/)

2015年にはクラウドのバックエンドサービスが充実するようです。

* プライベートクラウド
* デバイスのダッシュボード
* データ可視化

### spark-server

自分のクラウド上にSpark Cloudのオープンソース版を構築できる[spark-server](https://github.com/spark/spark-server)もあります。Spark Cloudと同じREST APIとSpark-CLIが使えるようです。しばらく更新されていなかったのですが今日コミットがたくさん入りました。これで安心して試すことができます。
