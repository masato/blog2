title: "IoTでアボカドの発芽促進とカビを防止する - Part1: リソース"
date: 2015-05-28 16:08:23
categories:
 - IoT
tags:
 - IoT
 - 電子工作
 - RaspberryPi
 - アボカド
description: 一通りセンサーやRaspberry Piとクラウド連係ができるようになったので、もう少し本格的なユースケースを考えています。最近気になっている農業とIoT連係を少しでも理解できるように手頃な観葉植物をモニタリングしてみようと思います。
---

一通りセンサーやRaspberry Piとクラウド連係ができるようになったので、もう少し本格的なユースケースを考えています。最近気になっている農業とIoT連係を少しでも理解できるように手頃な観葉植物をモニタリングしてみようと思います。


<!-- more -->

## 植物の水やり自動化

コネクテッドガーデンとかスマートガーデニングシステムとか呼ばれる分野です。植物の水やりを自動化をする仕組みを作って遊んでいる人が多いみたいです。スマホをタップしたり、Slackにつぶやくことでリモートの植物に水やりをすることができます。

* [Automatic garden watering system with Raspberry Pi and Arduino Uno Rev3](http://martinsund.se/2015/05/22/automatic-garden-watering-system-with-raspberry-pi-and-arduino-uno-rev3/)
* [Automatically watering your plants with sensors, a Pi and webhooks](https://blog.serverdensity.com/automatically-watering-your-plants-with-sensors-a-pi-and-webhooks/)
* [Raspberry PI と Hubot で観葉植物の水やりを自動化する](http://ja.ngs.io/2014/08/02/watering-pi/)
* [Arduinoで自動水やり器を作る①](http://qiita.com/interestor/items/d59590b64820a8cf973e)


## 農業センサー

世界の農業問題や食糧供給量、水問題を解決するために、効率的に植物が成長するためのデータを集める取り組みもあります。市販してITに詳しくない普通の人に使ってもらうためには、工業製品として扱いやすくすぐに壊れないような品質、安価で入手しやすいことが重要になります。Raspberry Piで遊んでいる自分とはちょっと見ている世界が違います。


* [世界の水問題を解決するスマート農業センサー「SenSprout」が注目を集める理由](http://thebridge.jp/2015/01/sensprout)
* [スマートガーデンシステム「Edyn」--環境変化の監視で植物の育成を支援](http://japan.cnet.com/news/service/35050334/)
* [Parrot Flower Power](http://www.parrot.com/jp/products/flower-power/)

## クラウド

電子工作から一歩進んでIoTやコネクテッドデバイスを楽しもうとすると、ArduinoやRaspberry Piをネットワークに接続してセンシングデータを送信したりメッセージを送受信する必要があります。複数のWebサービスを上手く連係するエージェントやボット、メッセージのブローカーをクラウド上に用意しようと思います。

* [Garduino Phone – Saving Plants from Negligent Parents using Twilio, Arduino and Sinatra](https://www.twilio.com/blog/2014/06/garduino-phone-using-twilio-arduino-and-sinatra.html)
* [Water your plants remotely using PiFace and Ubidots](http://blog.ubidots.com/water-your-plants-remotely-using-piface-and-ubidots)
 
## ダッシュボード

[Bug Labs](http://buglabs.net/)がIoT用のTwitterみたいなメッセージングサービスの[dweet.io](https://dweet.io/)とダッシュボードの[freeboard](http://freeboard.github.io/freeboard/)を無料で提供しています。[freeboard](https://github.com/Freeboard/freeboard)はGitHubに公開されているので自分でホスティングすることもできます。

* [Thirsty Plant Uses dweet™ to Alert Owner via Electric Imp](http://buglabs.tumblr.com/post/107608001031/thirsty-plant-uses-dweet-tm-to-alert-owner-via)
* [dweet.io + InfluxDB + Grafana](http://datadventures.ghost.io/2014/09/07/dweet-io-influxdb-grafana/)
* [Using DevOps Tools to Monitor a Polytunnel](http://blog.risingstack.com/using-devops-tools-to-monitor-polytunnel/)
* [Cool-looking dashboard for connected devices](https://community.particle.io/t/cool-looking-dashboard-for-connected-devices/4021)


## アボカドの水耕栽培

何か手頃な水耕栽培ができる観葉植物がないか調べているとアボカドの栽培が楽しそうです。アボカドを食べたあと残った種に爪楊枝を挿します。平らな方を下にして水を入れたガラス容器つけるだけです。


* [その種捨てちゃうの？ アボカドを観葉植物に育てる方法](http://matome.naver.jp/odai/2134978820939247201)
* [うそーん( ﾟДﾟ)家でできるの？簡単【アボカド栽培】](http://matome.naver.jp/odai/2134901203259988201)
* [アボカドやハーブを部屋で育てる!?簡単かわいい【水栽培】にトライしよう！](https://kinarino.jp/cat6-%E3%83%A9%E3%82%A4%E3%83%95%E3%82%B9%E3%82%BF%E3%82%A4%E3%83%AB/9984-%E3%82%A2%E3%83%9C%E3%82%AB%E3%83%89%E3%82%84%E3%83%8F%E3%83%BC%E3%83%96%E3%82%92%E9%83%A8%E5%B1%8B%E3%81%A7%E8%82%B2%E3%81%A6%E3%82%8B!%EF%BC%9F%E7%B0%A1%E5%8D%98%E3%81%8B%E3%82%8F%E3%81%84%E3%81%84%E3%80%90%E6%B0%B4%E6%A0%BD%E5%9F%B9%E3%80%91%E3%81%AB%E3%83%88%E3%83%A9%E3%82%A4%E3%81%97%E3%82%88%E3%81%86%EF%BC%81)
* [アボカドを水耕栽培して観葉植物にする](http://nijimo.jp/blog/hobby/2015/1304/)


### センサーやLEDライト

水温、気温、湿度、気圧を測ってアボカド周辺の環境データを取得します。部屋の中で育てるのでLEDライトも必要かも知れません。

* [BME280](https://www.switch-science.com/catalog/2236/)
* [DS18B20防水仕様](http://victory7.com/?pid=65664796)
* [植物育成LEDライト](http://www.amazon.co.jp/dp/B00LTKSI5O/)
