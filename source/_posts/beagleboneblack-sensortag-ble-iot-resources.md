title: "Node.jsでつくるIoT - Part2: Beagle Bone BlackとSensorTagとBLE"
date: 2015-01-25 00:53:15
tags:
 - IoT
 - BLE
 - SensorTag
 - BeagleBoneBlack
 - InfluxDB
 - Riemann
 - Dashboard
 - freeboard
 - PubNub
 - Bluemix
 - IBMIoTFoundation
description: UbuntuをインストールしたBeagle Bone Blackもしばらく放置していました。Connected Hardwareな感じが薄れますが、IoTも最初はLinuxが動くマシンを使うと敷居が低く入りやすいです。そこでBeagle Bone BlackをIoTのConnected Hardwareとして使っていこうと思います。まずは良い事例やチュートリアルがないか調べます。
---

Ubuntuをインストールした[Beagle Bone Black](http://beagleboard.org/black)もしばらく放置していました。Connected Hardwareな感じが薄れますが、IoTも最初はLinuxが動くマシンを使うと敷居が低く入りやすいです。そこでBeagle Bone BlackをIoTのConnected Hardwareとして使っていこうと思います。まずは良い事例やチュートリアルがないか調べます。

<!-- more -->

## 参考サイト

[SensorTag](http://www.tij.co.jp/ww/wireless_connectivity/sensortag/index.shtml)とBluetooth LE使ったサンプルをいくつか探しました。

* [BlueMix Internet of Things Workshop with Texas Instruments BeagleBone and SensorTag](https://deskinhursley.wordpress.com/2014/05/20/bluemix-internet-of-things-workshop-with-texas-instruments-beaglebone-and-sensortag/)
* [BeagleBone with SensorTag](https://developer.ibm.com/iot/recipes/ti-beaglebone-sensortag/)
* [Bluetooth Low Energy on BeagleBone Black](http://blog.revealinghour.in/bluetooh-low-energy/2014/06/05/bluetooth-low-energy-on-beaglebone-black/)
* [Using DevOps Tools to Monitor a Polytunnel](http://blog.risingstack.com/using-devops-tools-to-monitor-polytunnel/)

[IBM IoT Foundationのレシピ](https://developer.ibm.com/iot/)はチュートリアルのページとBluemixが連携しているので、アカウントを持っている人にはとても便利な仕組みです。こういったレシピは最近よく見かけますが情報共有にとても有効です。[Riemann+InfluxDB+Grafanaの組み合わせ](/2014/09/26/docker-monitoring-stack-prepare/)は気に入っているので、Raspberry Piを使ったビニルハウス監視を参考にしてみます。


## デバイス側の準備

Beagle Bone BlackにはタイプAのUSBポートが一つしかありません。現在は無線LANドングルを挿して使っています。SensorTagとBLEで通信するためUSBポートを空けたいので、インターネット接続はWindowsやOSXとUSBシリアル接続経由で行います。USBシリアル接続はBeagle Bone Blackの電源供給も兼ねるので開発用途には便利ですが、単独で使えないので設置場所が限られてしまいます。とりあえずSensorTagとBLEの練習用です。


### Ubuntu 14.04.1のインストール

現在はBeagle Bone Blackに[Ubuntu 13.10をインストール](/2014/05/03/beagleboneblack-ubuntu/)しているので、新しいイメージを再インストールします。

### SensorTagの購入

Texas Instrumentsの[TI Store](http://www.tij.co.jp/ww/wireless_connectivity/sensortag/index.shtml)から、SensorTagを購入します。送料無料というブログもありましたが、今購入すると送料は$20.00かかります。FAQの[Why are you now charging shipping and handling?](http://www.ti.com/lsds/ti/store/faq-ti-store-orders.page#shippingHandling)によると送料無料はコストに合わなくなったのでやめたそうです。合計$45.00でした。

### Bluetooth LE USBドングルの購入

Raspberry PiやBeagle Bone Blackで実績があるらしい、Planexの[BT-Micro4](http://www.amazon.co.jp/dp/B0071TE1G2)をAmazonから購入します。1,343円です。2015-03-20まで「ビデオレンタルで使える200円クーポンプレゼント」のキャンペーン中です。

## バックエンド側の準備

### DIYの場合

今のところIoTデータストアにはInfluxDBにする予定です。[以前用意した](/2014/09/26/docker-monitoring-stack-prepare/)Riemann、InfluxDB、Grafana用のDockerコンテナを使います。

### クラウドサービスを使う場合

[PubNub](http://www.pubnub.com/)のブログから[freeboard](https://freeboard.io/)をダッシュボードに使ったサンプルを見つけました。

* [Realtime IoT Monitoring for Devices with PubNub Presence - See more at: http://www.pubnub.com/blog/realtime-monitoring-of-internet-of-things-devices/#sthash.rdruILX5.dpuf](http://www.pubnub.com/blog/realtime-monitoring-of-internet-of-things-devices/)

PubNubもfreeboardもFREEプランがあるので、サーバーサイドは自分で構築しなくても使えそうです。
