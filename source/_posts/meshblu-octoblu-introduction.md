title: "Node.jsでつくるIoT - Part3: OctobluのMeshblu"
date: 2015-03-16 22:49:08
tags:
 - Octoblu
 - Meshblu
 - Nodejs
 - Cylonjs
 - IoT
 - MQTT
description: Octobluは2014年に設立されたIoT事業者です。同じ年の12月にはCitrixに買収されています。IoTプラットフォームのMeshbluでは、コネクテッドデバイスと人やWebサービスをリアルタイム通信でつなぐためのプラットフォームと周辺技術の開発をオープンソースで行っています。ソースコードはMITライセンスでGitHubに公開されています。また、このプラットフォームを使ったサービスも現在ベータ版で提供を開始しています。
---

[Octoblu](http://octoblu.com/)は2014年に設立されたIoT事業者です。同じ年の12月には[Citrixに買収](http://thenewstack.io/citrix-acquires-octoblu-the-drone-networking-company-formerly-known-as-skynet/)されています。IoTプラットフォームの[Meshblu](http://developer.octoblu.com/)では、コネクテッドデバイスと人やWebサービスをリアルタイム通信でつなぐためのプラットフォームと周辺技術の開発をオープンソースで行っています。ソースコードはMITライセンスで[GitHub](https://github.com/octoblu)に公開されています。また、このプラットフォームを使ったサービスも現在[ベータ版](https://app.octoblu.com/invitation/request)で提供を開始しています。

<!-- more -->

## Pub/Subのマルチプロトコル対応

以前Skynetと呼ばれていたOctobluのプラットフォームはMeshbluに名称変更されました。Meshbluは[Ponte](https://eclipse.org/ponte/)と同じように、HTTP、WebSpocket、MQTT、CoAPといったマルチプロトコルに対応したブローカーを実装しています。MQTTブローカーは[Mosca](https://github.com/mcollina/mosca)なので馴染み深いです。異なるプロトコル同士でPub/Subができると、curlでpublishしてMQTT.jsでsubscribeするとかコネクテッドデバイスとのメッセージ交換が楽しくなります。

## データストア

コネクテッドデバイスのセンサーデータをRedisやMongoDBにストアするAPIもあります。簡単なREST APIのクエリ文字列使って最新の5件を取得したり、期間指定もできます。ストリームにも対応しているのでビジュアライゼーションに重宝しそうです。

## デバイス/ノードの識別

デバイスやサービスのノードはuuidとtokenで識別されます。登録済みのデバイスやオーナーに指定したデバイス間でのみメッセージの送受信ができます。MQTTのトピック名の扱いがちょっと特殊なので戸惑いますが、異なるプロトコルでメッセージを交換するためのよく考えられた仕様だと思います。

## Node.js/JavaScriptのSDK

Node.jsやJavaScriptのライブラリが充実しています。[Cylon.jsのアダプタ](http://cylonjs.com/documentation/platforms/skynet/)を使うと簡単にコネクテッドデバイスからセンサーデータを収集したり、イベント駆動なアプリを動かすことができます。ロジックの多くはクラウド上でNode.jsで実装したいので、コネクテッドデバイスからセンサーデータを取得するためのコードを[Cylon.js](http://cylonjs.com/)や[Johnny-Five](https://github.com/rwaldron/johnny-five)で書けると便利です。

ただし、以前のSkynetのパッケージ名が残っていたり、複数あるライブラリによってプロパティ名が異なっていたり、最初は少し混乱します。過渡期みたいなのでもう少しすれば整理されると思います。

## コミュニティ

[hackster.io](http://www.hackster.io/octoblu)に[コミュニティページ](http://www.hackster.io/octoblu)があるのでコネクテッドデバイスとクラウドを連携したサンプルをいろいろ試すことができます。