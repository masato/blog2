title: "源ノ角ゴシック"
date: 2014-08-02 02:50:07
tags:
 - font
 - Hexo
 - NitrousIO
description: AdobeとGoogleが共同で3年もかけて制作した、源ノ角ゴシックがオープンソースでリリースされました。"ゲンノカクゴシック"と読みます。"ノ"がジブリメソッドらしいです。さっそくNitrous.IO+Hexo+bitureのブログでも使ってみました。
---

AdobeとGoogleが共同で3年もかけて制作した、源ノ角ゴシックがオープンソースでリリースされました。"ゲンノカクゴシック"と読みます。
"ノ"が[ジブリメソッド](http://ascii.jp/elem/000/000/917/917366/index-5.html)らしいです。

さっそくNitrous.IO+Hexo+bitureのブログでも使ってみました。


### インストール

[Google Noto Fonts](http://www.google.com/get/noto/#/family/noto-sans-jpan)から`Noto Sans Japanese`をダウンロードします。

``` bash
$ cd
$ wget http://www.google.com/get/noto/pkgs/NotoSansJapanese-windows.zip
$ unzip NotoSansJapanese-windows.zip
$ mkdir ~/workspace/blog/themes/biture/source/fonts
$ cp NotoSansJapanese-windows/NotoSansJP-Thin-Windows.otf ~/workspace/blog/themes/biture/source/fonts
```

スタイルシートにfont-familyを追加します。
ウェイトとサイズは悩ましいのですが、DemiLightと16pxにしました。

``` css ~/workspace/blog/themes/biture/source/css/biture.css

body {
...
  font-size: 16px;
...
}
...
@font-face {
font-family: "Noto Sans";
  src: url(/fonts/NotoSansJP-DemiLight.otf) format("opentype");
}
...
.article-entry {
  margin: 15px 0 35px 0;
  text-align:justify !important;
  font-family: "Noto Sans", Verdana, Roboto, "Droid Sans", "游ゴシック", YuGothic, "ヒラギノ角ゴ ProN W3", "Hiragino Kaku Gothic ProN", "メイリオ", Meiryo, sans-serif;
}
```