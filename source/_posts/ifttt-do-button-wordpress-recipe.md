title: "IFTTTのDo Buttonアプリを使いBeagleBoneBlackへ位置情報のGoogleマップを送る"
date: 2015-02-21 13:14:47
tags:
 - IFTTT
 - DoButton
 - Nodejs
 - MQTT
 - express-ifttt-webhook
 - Yo
description: 先日IFTTTからDoアプリがリリースされました。Do Button、Do Camera、Do Noteの3つあり、iOS と Androidから使えるようになっています。すでにWordPressを使ったCreate a photo post with a map of your locationというDoレシピがあるので使ってみようと思います。
---

先日IFTTTから[Doアプリ](http://blog.ifttt.com/post/111467477713/introducing-do-a-new-class-of-apps-by-ifttt)がリリースされました。Do Button、Do Camera、Do Noteの3つあり、iOS と Androidから使えるようになっています。すでにWordPressを使った[Create a photo post with a map of your location](https://ifttt.com/recipes/259548-create-a-photo-post-with-a-map-of-your-location)というDoレシピがあるので使ってみようと思います。

<!-- more -->

## Do Button

これまでのレシピはIFレシピのタブにまとめられ、新たにDoレシピのタブが追加されました。Do Buttonの動作は簡単です。Yoトリガーを使うようにシンプルにAndroidアプリのボタンを押すだけです。ちょうどIFTTTとYoを連携させたレシピを書いて使っているところなので、Do Button用にレシピを作成します。

## Do WordPress

### Yoトリガーとexpress-ifttt-webhookアクションのサンプル

[前回](/2015/02/17/express-ifttt-webhook-requestbin/)Yoトリガーを使ったIFレシピのサンプルを作りました。[WordPress Channel](https://ifttt.com/wordpress)のXML-RPCを借りて、クラウド上に構築したNode.js製の[express-ifttt-webhook](https://github.com/b00giZm/express-ifttt-webhook)ゲートウェイに向けてアクションを実行します。ゲートウェイではMQTTにブリッジしてpublishします。結果としてYoからMQTTでsubscribeしているローカルのBegleBone Blackへ任意のメッセージを送ることができます。

### Do レシピ

[Create a photo post with a map of your location](https://ifttt.com/recipes/259548-create-a-photo-post-with-a-map-of-your-location)のDoレシピは、AndroidのDo Buttonアプリのボタンを押すと、モバイルの位置情報からGoogleマップの位置情報を取得してWordPressにポストしてくれます。

![do-recipe-wordpress](/2015/02/21/ifttt-do-button-wordpress-recipe/do-recipe-wordpress.png)

### WordPress Channel

Do WordPressレシピのページを開きます。WordPressのアイコンをクリックするとWordPress Channelの構成画面に移動します。[前回](/2015/02/17/express-ifttt-webhook-requestbin/)すでにチャンネルは構成済みなので変更はありません。Blog URLはクラウドの仮想マシン上に構築した[express-ifttt-webhook](https://github.com/b00giZm/express-ifttt-webhook)のURLを指定します。

![wordpress-channel](/2015/02/21/ifttt-do-button-wordpress-recipe/wordpress-channel.png)

Do WordPressレシピのページに戻り`Add Recipe`ボタンをクリックしてレシピを追加します。レシピページのActionはデフォルトで動作しますが適宜フィールドに入力して`Update`ボタンをクリックします。


## テスト

### BeagleBone BlackのCylon.jsアプリ

BeagleBone BlackでMQTTをsubsribeするプログラムは前回と同じです。単純にメッセージを受信して標準出力します。

```js app.js
/*eslint-env node */

var Cylon = require('cylon');

Cylon.robot({
  connections: {
    server: { adaptor: 'mqtt', host: 'mqtt://xxx.xxx.xxx.xxx:1883' },
    beaglebone: { adaptor: 'beaglebone' }
  },
  
  work: function(my) {
    my.server.subscribe('ifttt/bbb');
    my.server.on('message', function (topic, data) {
      console.log(topic + ": " + data);
    });
  }
}).start();
```

`npm start`を実行してアプリを起動します。

``` bash
$ npm start

> cylon-test@0.0.1 start /home/ubuntu/node_modules/orion/.workspace/cylonjs
> node app.js

I, [2015-02-21T05:01:32.480Z]  INFO -- : Initializing connections.
I, [2015-02-21T05:01:34.608Z]  INFO -- : Initializing devices.
I, [2015-02-21T05:01:34.621Z]  INFO -- : Starting connections.
I, [2015-02-21T05:01:34.681Z]  INFO -- : Starting devices.
I, [2015-02-21T05:01:34.684Z]  INFO -- : Working.
```

### AndroidのDo Buttonアプリ

AndroidのDo Buttonアプリを起動すると、先ほど追加したDo WordPressが追加されています。中央のWordPressロゴをタップするとアプリが実行されます。BeagleBone BlackのコンソールにGoogleマップのURLとIFTTTのシェアページの短縮URLが通知されました。

![do-button-android](/2015/02/21/ifttt-do-button-wordpress-recipe/do-button-android.png)

``` bash
> cylon-test@0.0.1 start /home/ubuntu/node_modules/orion/.workspace/cylonjs
> node app.js

I, [2015-02-21T05:01:32.480Z]  INFO -- : Initializing connections.
I, [2015-02-21T05:01:34.608Z]  INFO -- : Initializing devices.
I, [2015-02-21T05:01:34.621Z]  INFO -- : Starting connections.
I, [2015-02-21T05:01:34.681Z]  INFO -- : Starting devices.
I, [2015-02-21T05:01:34.684Z]  INFO -- : Working.
ifttt/bbb: {"username":"username","password":"password","title":"","description":"<div><img src='http://maps.google.com/maps/api/staticmap?center=35.xx,139.xxx&zoom=19&size=640x440&scale=1&maptype=roadmap&sensor=false&markers=color:red%7C35.xxx,139.xxx' style='max-width:600px;' /><br/><div>Do Button pressed on February 21, 2015 at 02:04PM http://ift.tt/1EltQE2</div></div>","tags":{"string":"Do Button"},"post_status":"publish"}
```
