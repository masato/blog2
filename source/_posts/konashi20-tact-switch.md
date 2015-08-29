title: "konashi 2.0 入門 - Part3: タクトスイッチ(S1)を押す"
date: 2015-08-24 20:53:15
categories:
 - IoT
tags:
 - konashi
 - JavaScript
 - Zeptojs
 - jsdoit
 - Lチカ
description: konashi.jsはjsdo.itと連動して使うことで、コミュニティが共有しているコードをForkして動作を確認しながら学習することができます。前回はまずはLチカ（LEDチカチカをForkして動かしました。次にLEDをコントロールしちゃおう☆をForkします。コードを編集して理解しながらタクトスイッチのON/OFFの状態を画面に表示するプログラムを書いてみます。
---

konashi.jsは[jsdo.it](http://jsdo.it)と連動して使うことで、コミュニティが共有しているコードをForkして動作を確認しながら学習することができます。[前回](/2015/08/22/konashi20-koshian/)は[まずはLチカ（LEDチカチカ)](http://jsdo.it/monakaz/nOMl)をForkして動かしました。次に[LEDをコントロールしちゃおう☆](http://jsdo.it/monakaz/w1gz)をForkします。コードを編集して理解しながらタクトスイッチのON/OFFの状態を画面に表示するプログラムを書いてみます。

<!-- more -->


## はじめての開発

jsdo.itの[konashiタグ](http://jsdo.it/tag/konashi)などから参考にしたいリポジトリをさがしてForkします。今回は[LEDをコントロールしちゃおう☆](http://jsdo.it/monakaz/w1gz)使わせていただきます。jsdo.itのエディタを使い、今回使わないコードの削除や編集していきます。

## JavaScript

オリジナルのLEDをコントロールするコードを削除して、タクトスイッチだけ動作するコードにします。[Zepto.js](http://zeptojs.com/)(jQuery互換)を使ったコードなので、だいたい何をしているか理解できると思います。イベントハンドラはJavaScrptによくある非同期処理で書いていきます。



今回は以下の4つのAPIにイベントハンドラを登録して処理を実装します。


* [find](http://konashi.ux-xu.com/documents/#base-find)
 * iPhone周辺のkonashiを検索する

* [ready](http://konashi.ux-xu.com/documents/#javascript-ready)
 * このイベントハンドラからkonashiと通信が可能になる

* [updatePioInput](http://konashi.ux-xu.com/documents/#javascript-updatePioInput)
 * PIOの入力の状態が変化したら実行される

* [digitalRead](http://konashi.ux-xu.com/documents/#pio-digitalRead)
 * PIOの特定のピンの入力状態を取得する
 * (事前にpinModeやpinModeAllでピンのモードを入力にしておく)


```js
// forked from monakaz's "LEDをコントロールしちゃおう☆" http://jsdo.it/monakaz/w1gz
$(function(){

  $("#btn-find").on("tap", function() {
    if($("#btn-find").hasClass("find")) {
      k.find();
    } else {
      k.disconnect();

      // change find button
      $("#btn-find")
        .addClass("find")
        .html("Find konashi");

      $("#s1-status").html("OFF");
    }
  });

  // konashiと接続できた
  k.ready(function() {
    // change find button
    $("#btn-find")
      .removeClass("find")
      .html("Disconnect konashi");

    k.pinMode(k.S1, k.INPUT);
  });

  // 入力が変化した時
  k.updatePioInput(function(data) {
    k.digitalRead(k.S1, function(value) {
      //k.log(value);
      if(value == k.HIGH) {
         $("#s1-status").html("ON");
      } else {
         $("#s1-status").html("OFF");
      }
    });
  });

  //k.showDebugLog();
});
```

また`k.showDebugLog();`をアンコメントするとkonashi.jsの左上にデバッグ用の領域が表示されます。`k.log(value);`のようにしてデバッグしたい値を出力することができます。


## HTML

HTMLファイルも今回利用しないLEDをコントロールする`li`要素は削除します。

```html
<!DOCTYPE html>
<html>
    <head>
        <meta name="viewport" content="width=device-width, initial-scale=2, minimum-scale=1, maximum-scale=1, user-scalable=no">

        <!-- ratchet css -->
        <link rel="stylesheet" href="http://jsrun.it/assets/h/F/P/P/hFPPa">

    </head>
    <body>
        <header class="bar-title">
            <h1 class="title">konashi.js: Drive PIO</h1>
        </header>

        <div class="content">

            <div class="hello">
                <p>Hello konashi.js!</p>
                <p>First make sure you tap the following button to find konashi.
            </div>

            <div class="find">
                <a id="btn-find" class="button-main button-block find">Find konashi</a>
            </div>

            <ul class="list inset">
                <li class="list-divider">PIO: Input Status</li>
                <li>S1: <span id="s1-status">OFF</span></li>
            </ul>

        </div>

        <!-- for konashijs -->
        <script src="http://konashi.ux-xu.com/kjs/konashi-bridge.min.js"></script>

        <!-- for this sample -->
        <!-- zepto -->
        <script src="http://jsrun.it/assets/1/M/0/f/1M0fl"></script>
        <!-- touch.js -->
        <script src="http://jsrun.it/assets/g/s/1/M/gs1MI"></script>
        <!-- ratchet.js -->
        <script src="http://jsrun.it/assets/g/3/W/u/g3WuF"></script>
    </body>
</html>
```


### CSS

オリジナルでは`display: none`の非表示にして、konashiに接続後に要素を表示しています。今回は最初から`ul`要素は表示します。id属性のスタイルも削除します。

```css
.hello {
    margin: 10px;
}

.find {
    margin: 10px;
}
```


## テスト

### コード検索

iPhoneからkonashi.jsアプリを起動します。アプリ内のブラウザからjsdo.itに作成したリポジトリを検索して表示します。「タクトスイッチを押してみる」をタップします。

![konashi-switch.png](/2015/08/24/konashi20-tact-switch/konashi-switch.png)

### コードのページ

コードのページの左側プレビューをタップします。

![konashi-switch-preview.png](/2015/08/24/konashi20-tact-switch/konashi-switch-preview.png)

### konashiの検索

「Find konashi」からkohashiを検索します。「Select Module」に表示されるkonashiタップします。

![konashi-switch-find.png](/2015/08/24/konashi20-tact-switch/konashi-switch-find.png)

### タクトスイッチを押す

konashiの本体にあるタクトスイッチを押すと、konashi.jsアプリのブラウザに表示されたテキストが「S1:OFF」から「S1:ON」に変更されます。




