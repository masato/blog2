title: "konashi 2.0 入門 - Part4: Framework7でネイティブ風に"
date: 2015-09-15 21:11:22
categories:
 - IoT
tags:
 - konashi
 - Framework7
 - Lチカ
 - RawGit
 - CDN
description: jsdo.itで作成したkonashi.jsアプリの画面はZepto.jsを使っていました。ちょっとシンプルなので最近HTML5モバイルアプリの開発で使っているFramework7を使ってLチカとタクトスイッチの画面を作ります。
---


[jsdo.it](http://jsdo.it/)で作成したkonashi.jsアプリの画面は[Zepto.js](http://zeptojs.com/)[touch.js](https://github.com/madrobby/zepto/blob/master/src/touch.js)を使っていました。ちょっとシンプルなので最近HTML5モバイルアプリの開発で使っている[Framework7](http://www.idangero.us/framework7/)を使ってLチカとタクトスイッチの画面を作ります。


<!-- more -->

## Framework7

[Framework7](http://www.idangero.us/framework7/#.VfleBiDtmko)は、HTML5/JavaScript/CSSといったWebアプリ開発でおなじみのセットを使いiOSやMaterialデザインのネイティブ風なモバイルアプリが開発できるフレームワークです。DOM操作は[Dom7](http://www.idangero.us/framework7/docs/dom.html)という同梱されているライブラリを使えます。ほぼjQuery/Zepto.jsと同じように`$$`を通して使うことができます。


## RawGitをCDNに使う

有名なJavaScriptのライブラリはGoogleの[Hosted Libraries](https://developers.google.com/speed/libraries/)などのCDNを通してHTMLからロードすることができます。残念ながらFramework7をCDNから見つけることができなかったので、[RawGit](https://rawgit.com/)を使いGitHubのリポジトリから直接ロードできるようにします。

[Can I run HTML files directly from GitHub, instead of just viewing their source?](http://stackoverflow.com/questions/6551446/can-i-run-html-files-directly-from-github-instead-of-just-viewing-their-source)を参考にして使ってみます。

[RawGit](https://rawgit.com/)をブラウザで開きテキストボックスに公開したいGtHub上のファイルを貼り付けるだけで、HTMLからロードできるURLを作成してくれます。

![rawgit.png](/2015/09/15/konashi20-framework7/rawgit.png)

### Framework7のURL

例えば、Framework7の`framework7.min.js`を使いたい場合GitHub上の以下のURLを一番上のテキストボックスに貼り付けます。

https://github.com/nolimits4web/Framework7/blob/master/dist/js/framework7.min.js

右側の開発環境用のテキストボックスに以下のURLが作成されます。

https://rawgit.com/nolimits4web/Framework7/master/dist/js/framework7.min.js

今回必要なURLは以下の3つです。

* framework7.min.js
https://rawgit.com/nolimits4web/Framework7/master/dist/js/framework7.min.js

* framework7.ios.min.css
https://rawgit.com/nolimits4web/Framework7/master/dist/css/framework7.ios.min.css

* framework7.ios.colors.min.css
https://rawgit.com/nolimits4web/Framework7/master/dist/css/framework7.ios.colors.min.css


## コード

今回作成したjsdo.itのコードは[こちら](http://jsdo.it/ma6ato/ItiZ)です。

[LEDをコントロールしちゃおう☆](http://jsdo.it/monakaz/w1gz)をForkしてFramework7用にコードを編集します。

### HTML

Framework7の基本のHTMLレイアウトは[こちら](http://www.idangero.us/framework7/docs/app-layout.html)のドキュメントページにあります。

CSSとJavaScriptのロードは上記の[RawGit](https://rawgit.com/)で作成したURLを使います。また、各コンポーネントのドキュメントもちゃんと用意されているので使いやすいです。

* ボタン: [Buttons](http://www.idangero.us/framework7/docs/buttons.html)
* リスト: [List View](http://www.idangero.us/framework7/docs/list-view.html)
* スイッチ: [Form elements](http://www.idangero.us/framework7/docs/form-elements.html)

```html
<!DOCTYPE html>
<html>
    <head>
        <meta charset="utf-8">
        <meta name="viewport" content="width=device-width, initial-scale=1, maximum-scale=1, minimum-scale=1, user-scalable=no, minimal-ui">
        <meta name="apple-mobile-web-app-capable" content="yes">
        <meta name="apple-mobile-web-app-status-bar-style" content="black">

        <!-- Framework7 css -->
        <link rel="stylesheet" href="https://rawgit.com/nolimits4web/Framework7/master/dist/css/framework7.ios.min.css">
        <link rel="stylesheet" href="https://rawgit.com/nolimits4web/Framework7/master/dist/css/framework7.ios.colors.min.css">
        <title>myThings F7</title>
    </head>
    <body>
        <div class="statusbar-overlay"></div>
        <div class="panel-overlay"></div>
        <div class="panel panel-left panel-reveal">
            <div class="content-block">
                <p>Left panel content goes here</p>
            </div>
        </div>
        <div class="views">
            <div class="view view-main">
                <div class="navbar">
                    <div class="navbar-inner">
                        <div class="center sliding">myThings F7</div>
                        <div class="right">
                            <a href="#" class="link icon-only open-panel"><i class="icon icon-bars"></i></a>
                        </div>
                    </div>
                </div>
                <div class="pages navbar-through toolbar-through">
                    <div data-page="index" class="page">
                        <div class="page-content">
                            <div class="content-block">
                                <a href="#" id="btn-find" class="find button button-big">Find konashi</a>
                            </div>
                            <div id="pio-setting">
                                <div class="content-block-title">PIO: Output Settings</div>
                                <div class="list-block">
                                    <ul>
                                        <li class="item-content">
                                            <div class="item-media"><i class="icon icon-form-toggle"></i></div>
                                            <div class="item-inner">
                                                <div class="item-title label">LED2</div>
                                                <div class="item-input">
                                                    <label class="label-switch">
                                                        <div class="toggle" data-pin="1">
                                                            <input type="checkbox">
                                                            <div class="checkbox"></div>
                                                        </div>
                                                    </label>
                                                </div>
                                            </div>
                                        </li>
                                    </ul>
                                </div>
                                <div class="content-block-title">PIO: Input Settings</div>
                                <div class="list-block">
                                    <ul>
                                        <li class="item-content">
                                            <div class="item-media"><i class="icon icon-form-settings"></i></div>
                                            <div class="item-inner">
                                                <div class="item-title label">S1</div>
                                                <div class="item-after" id="s1-status">OFF</div>
                                            </div>
                                        </li>
                                    </ul>
                                </div>
                            </div>
                        </div>
                    </div>
                </div>
            </div>
        </div>
        <!-- for Framework7 -->
        <script src="https://rawgit.com/nolimits4web/Framework7/master/dist/js/framework7.min.js"></script>
        <script src="http://konashi.ux-xu.com/kjs/konashi-bridge.min.js"></script>
    </body>
</html>
```

### JavaScript

参考にした[コード](http://jsdo.it/monakaz/w1gz)はjQuery互換の[Zepto.js]((http://zeptojs.com/)を使って`$`変数からDOM操作をしていました。Framework7の[Dom7](http://www.idangero.us/framework7/docs/dom.html)は`$`を`$$`に置き換えるとそのまま使えました。

コンポーネントは[touch.js](https://github.com/madrobby/zepto/blob/master/src/touch.js)とAPIが異なっているのでほぼ書き直しています。

```js
(function(Framework7, $$){
  $$('.toggle').on("click", function(e){
    var pin = $$(e.currentTarget).data("pin");
    var value = $$('input').prop('checked') ? k.HIGH : k.LOW;
    k.digitalWrite(pin, value);
  });

  $$("#btn-find").on("click", function(){
    if($$("#btn-find").hasClass("find")){
      k.find();
    } else {
      k.disconnect();

      // change find button
      $$("#btn-find")
        .addClass("find")
        .html("Find konashi")
      ;

      // hide pio list
      $$("#pio-setting").hide();
      $$("#s1-status").html("OFF");
    }
  });

  k.on("ready", function(){
    // change find button
    $$("#btn-find")
      .removeClass("find")
      .html("Disconnect konashi");

    // show pio list
    $$("#pio-setting").show();

    k.pinModeAll(254);
  });

  k.updatePioInput( function(data){
    if(data % 2){
      $$("#s1-status").html("ON");
    } else {
      $$("#s1-status").html("OFF");
    }
  });

  //k.showDebugLog();
})(Framework7, Dom7);
```

### CSS

konashiが見つかるまでLチカやタクトスイッチの操作画面を非表示にするためのスタイルです。

```css
#pio-setting {
    display: none;
}
```

## 使い方

iPhoneに[konashi.js](https://itunes.apple.com/jp/app/konashi.js-javascript-html/id652940096)アプリをインストールして起動します。

### jsdo.it コード検索

検索から「Framework7でLチカ」を検索して実行します。

![konashi-f7.png](/2015/09/15/konashi20-framework7/konashi-f7.png)

### Framework7でLチカ

左側に表示されているプレビューをタップします。

![konashi-f7-select.png](/2015/09/15/konashi20-framework7/konashi-f7-select.png)

### Find Konashi

「Find Konashi」ボタンをタップして自分のdkonashiを選択します。

![konashi-f7-find.png](/2015/09/15/konashi20-framework7/konashi-f7-find.png)

### INPUTとOUTPUTを操作する

INPUTのLED2スイッチをONにすると、konashiのLED2が点灯します。konashiのタクトスイッチを押したり離したりするとS1の画面表記がON/OFFと変化します。

![konashi-f7-pio.png](/2015/09/15/konashi20-framework7/konashi-f7-pio.png)
