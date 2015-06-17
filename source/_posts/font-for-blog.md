title: 'ブログ用のフォント'
date: 2014-05-07 20:23:10
tags:
 - Hexo
 - font
description: Hexoのブログでは、テーマにbitureを使っています。bitureは、Open Sansや Helvetica Neueをスタイルで指定していますが、Androidから見ることも考慮して、きれいな日本語フォントを探しました。
---
Hexoのブログでは、テーマに[biture](https://github.com/kywk/hexo-theme-biture)を使っています。
bitureは、`Open Sans` や `Helvetica Neue`をスタイルで指定していますが、Androidから見ることも考慮して、きれいな日本語フォントを探しました。
<!-- more -->
マルチデバイスでも統一してきれいな日本語が表示できるように、[CSSでのフォント指定について考える（2014年）](http://www.dtp-transit.jp/misc/web/post_1881.html)を参考にしてCSSをカスタマイズしました。

### sans-serif
ここで紹介されている、フルスペックのsans-serifです。
``` css
font-family: Verdana, Roboto, "Droid Sans", "游ゴシック", YuGothic, "ヒラギノ角ゴ ProN W3", "Hiragino Kaku Gothic ProN", "メイリオ", Meiryo, sans-serif;
```
[Verdana](http://ja.wikipedia.org/wiki/Verdana)は、WindowsとOSXにインストールされているWebセーフフォントです。[IKEA](http://typophile.com/node/61222)的な。
Windows用に、`"Meiryo UI"`を指定していたことが多かったですが、他のデバイスと比べるとやはり行間が狭いので、メイリオを使うことにします。
iOSには、`"Hiragino Kaku Gothic ProN"`が、Android 4+では、Robotoがインストールされています。

### CSSの修正
font-familyを日本語用に修正します。
``` css ~/workspace/blog/themes/biture/source/css/boture.css
body {
  width: 100%;
  margin: 0px;
  padding: 0px;
  background-color: #eee;
  text-align: center;
  /*font-family: 'Open Sans', 'Helvetica Neue', Helvetica, Arial, sans-serif;
  */
  font-family: Verdana, Roboto, "Droid Sans", "游ゴシック", YuGothic, "ヒラギノ角ゴ ProN W3", "Hiragino Kaku Gothic ProN", "メイリオ", Meiryo, sans-serif;
  
  font-size: 14px;
  
  color: #444;
}

h1, h2, h3, h4, h5, h6 {
  line-height: 1.5em;
 /*
  font-family: 'Varela Round', 'Helvetica Neue', Helvetica, Arial,Pr sans-serif;
 */
  font-family: 'Varela Round', Verdana, Roboto, "Droid Sans", "游ゴシック", YuGothic, "ヒラギノ角ゴ ProN W3", "Hiragino Kaku Gothic ProN", "メイリオ", Meiryo, sans-serif;

  font-weight: normal;
  text-rendering: optimizelegibility;
}
```
### まとめ
このブログではタイトルに、Webフォントの[Varela Round](https://www.google.com/fonts/specimen/Varela+Round)使っています。Googleフォントにもトレンドがあるので、定期的にフォント指定は見直していきます。

