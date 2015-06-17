title: "Chrome Dev Editor で Dart 1 for Everyone - Part1: Chapter 1"
date: 2014-08-30 02:26:02
tags:
 - ChromeDevEditor
 - Dart
 - Chromebook
 - NitrousIO
 - Polymer
description: なかなかHP Chromebook 11で遊ぶ時間がなかったのですが、Google IO 2014で公開されたChrome Dev Editor (CDE)(aka Spark)をようやく試してみます。CDEはDartで書かれたChromeアプリです。エディターとWebサーバーが内蔵されているので、CDEだけでコードを書いて実行することができます。DartやJavaScript、Polymerを使ってChromeアプリやWebアプリの学習環境に使えそうです。タイミングよくPragmatic Bookshelfから、Dart 1 for EveryoneのBeta eBookがリリースされたのでさっそく写経します。Chromebookでの開発はNitrous.IOが日本語も使えてかなり便利なので、ほとんどcroutonは使わなくなりました。CDEを使うと気軽にDartやPolymerアプリを書けるので、Chromebookがもっと遊べそうです。
---

なかなか`HP Chromebook 11`で遊ぶ時間がなかったのですが、`Google IO 2014`で公開された`Chrome Dev Editor (CDE)`(aka Spark)をようやく試してみます。

CDEはDartで書かれたChromeアプリです。エディターとWebサーバーが内蔵されているので、CDEだけでコードを書いて実行することができます。DartやJavaScript、Polymerを使ってChromeアプリやWebアプリの学習環境に使えそうです。

タイミングよく`Pragmatic Bookshelf`から、[Dart 1 for Everyone](https://www.pragprog.com/book/csdart1/dart-1-for-hipsters)の`Beta eBook`がリリースされたのでさっそく写経します。

Chromebookでの開発はNitrous.IOが日本語も使えてかなり便利なので、ほとんどcroutonは使わなくなりました。
CDEを使うと気軽にDartやPolymerアプリを書けるので、Chromebookがもっと遊べそうです。

<!-- more -->

### Chrome Dev Editorのインストール

Chromeウェブストアから`Chrome Dev Editor`をインストールするとアプリランチャーから起動できるようになります。

### プロジェクトの作成

`New Project`メニューからダイアログで必要な情報を入れてCREATEボタンを押します。

* Project name: your_first_dart_app
* Project type: Dart web app

`CHOOSE FOLDER`でGoogleドライブをルートフォルダに指定して配下にプロジェクトを作成していきます。

Dartのひな形ファイルがプロジェクトに作成されています。このままさっそくRunボタンを押してみます。

Chromeブラウザでタブが開き、"Hello, World!"が表示されました。

CDEにはDartVMが搭載されてnative実行されると思ったのですが、JavaScriptにコンパイルされます。
非力な`HP Chromebook 11`だとスペック的にちょっと厳しめです。


{% img center /2014/08/30/chromedeveditor-dart-1-chapter-1/helloworld.png %}


### Chapter 1

`Dart 1 for Everyone`のChapter 1を`Chrome Dev Editor`で写経していくのですが、このままでは動きませんでした。Beta版なので動くコードになっていないようです。これも勉強なので修正していきます。

### 完成したプロジェクト

`your_first_dart_app`プロジェクトのwebフォルダー配下にコードを書きます。

ひな形のindex.htmlに、ラベルとJSONファイルをレンダリングするCSSセレクターを配置します。
Dartファイルと、`dart.js`のブートストラップをロードします。

``` html web/index.html
<!DOCTYPE html>

<html>
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Dart Comics</title>
  <link rel="stylesheet" href="styles.css">
</head>

<body>
  <h1>Dart Comics</h1>
  <p>Welcome to Dart Comics</p>
  <ul id="comics-list"></ul>
  <script type="application/dart" src="main.dart"></script>
  <script src="packages/browser/dart.js"></script>
</body>
</html>
```

Dartのメインスクリプトです。`res.open()`でGETするURLを相対パスのcomics.jsonに変更します。
ローカルのJSONを`#comics-list`のul要素にレンダリングします。

``` dart web/main.dart
import 'dart:html';
import 'dart:convert';

main() {
  var list_el = document.query('#comics-list');
  var req = new HttpRequest();
  req.open('get', 'comics.json');
  req.onLoad.listen((req) {
    var list = JSON.decode(req.target.responseText);
    list_el.innerHtml = graphic_novels_template(list);
  });
  req.send();
}

graphic_novels_template(list) {
  var html = '';
  list.forEach((graphic_novel) {
    html += graphic_novel_template(graphic_novel);
  });
  return html;
}

graphic_novel_template(graphic_novel) {
  return '''
  <li id="${graphic_novel['id']}">
    ${graphic_novel['title']}
    <a href="#" class="delete">[delete]</a>
  </li>''';
}
```

テスト用にロードするJSONデータはせっかくなので日本語で書きました。
原作のコミックはウォッチメンしか読んだことがないです。

```json web/comics.json
[
  {"title":"ウォッチメン",
  "author":"アラン・ムーア",
  "id":1},
  {"title":"Vフォー・ヴェンデッタ",
  "author":"アラン・ムーア",
  "id":2}, 
  {"title":"サンドマン",
  "author":"ニール・ゲイマン",
  "id":3} 
]
```

スタイルシートはCDEのひな形をそのまま使います。

``` css web/styles.css
body {
  background-color: #F8F8F8;
  font-family: 'Open Sans', sans-serif;
  font-size: 18px;
  margin: 15px;
}
```

### 実行

非力な`HP Chromebook 11`で作業しているので、DartのJavaScriptコンパイルに数分かかります。
コードを書いてすぐブラウザで確認という用途には難しいです。

{% img center /2014/08/30/chromedeveditor-dart-1-chapter-1/dartcomics.png %}

### Nitrous.IOと比べて

CDEはローカルのChromeアプリ内に実行環境があり、ソースコードもローカルにおきます。
CDEからGitも使えそうですがよくわからないので、別環境のCDEへはGoogleドライブ経由でソースコードを同期しています。

Chromebookで都度JavaScriptのコンパイルはちょっと無理があるので、開発環境がクラウド上で完結するNitrous.IOのような`Cloud Editor`のほうが使い勝手がよいです。
