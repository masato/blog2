title: "LinuxMint17 MATEにPPAでDart Editorをインストールする"
date: 2014-06-04 12:19:17
tags:
 - LinuxMint17
 - Dart
 - DartEditor
 - IDE
 - TypeScript
 - NitrousIO
 - VisualStudioOnlineMonaco
description: 昨日、IDCFクラウド上にxrdpで接続するGUI開発環境の準備ができたので、Dart Editorをインストールして本格的にDartを勉強しようと思います。そんな時、Windows Azure WebsitesとVisual Studio Online Monacoを使うと、TypeScriptとNode.jsをすべてクラウド上で開発できるらしく、気になって仕方がないです。
---

昨日、IDCFクラウド上にxrdpで接続する[GUI開発環境の準備](/2014/06/02/idcf-linuxmint17-part2/)ができたので、
[Dart Editor](https://www.dartlang.org/tools/editor/)をインストールして本格的にDartを勉強しようと思います。

そんな時、[Windows Azure Websites](http://azure.microsoft.com/ja-jp/services/web-sites/)と[Visual Studio Online "Monaco"](http://blogs.msdn.com/b/monaco/)を使うと、
[TypeScript](http://www.typescriptlang.org/)とNode.jsをすべてクラウド上で開発できるらしく、気になって仕方がないです。


<!-- more -->

### PPAからインストール

Dartの[PPAリポジトリ](https://groups.google.com/a/dartlang.org/forum/#!topic/misc/V3u8oogUDxQ)があったので、apt-getしてみます。

``` bash
$ sudo add-apt-repository ppa:hachre/dart
$ sudo apt-get update
$ sudo apt-get install darteditor
```

zipを[ダウンロード](https://www.dartlang.org/tools/editor/)してもよいですが、なるべくパッケージマネージャを使いたいです。

### Dart Editorの起動
`Menu -> プログラミング -> Dart Editor`から起動します。ここにDartiumのアイコンも並んであります。

IDCFクラウド上のインスタンスにxrdpで接続するので、若干ひっかかりを感じますが、普通に開発できそうです。


### Dartiumが起動しない
pop_pop_winのサンプルプロジェクトを実行しようとすると、
`libudev.so.0: cannot open shared object`となってDartiumが起動しません。
[issues](https://code.google.com/p/dart/issues/detail?id=12325)を見るとworkaroundでシムリンクをはるようにあったので、従います。

``` bash
$ sudo ln -s /lib/x86_64-linux-gnu/libudev.so.1 /lib/x86_64-linux-gnu/libudev.so.0
```
そうるすと`Google APIキーが欠如しています`と警告がでますが、Dartimuが起動するようになりました。


### Chromeが起動しない
すると今度は、Dartiumと共存できないのか、Chromeが起動しなくなりました。
Chromeがプラットフォーム化してくると、純粋なブラウザに特化したモダンな軽量ブラウザが欲しくなります。

動きがあやしく、psをみたらdartのプロセスが複数あがっていました。
xrdで複数の端末から接続しているのですが、うまくセッションが閉じられていないようです。
Chromeの日本語入力がおかしかったのも、このためか、psで別プロセスがないことを確認すると、入力できるようになりました。


### まとめ

DartVMを捨ててdart2jsだけにして、JavaScriptへコンパイルする言語としてみると、Dartは非常に魅力的です。
AngularDartなど触っていると、ああこれがモダンな静的型付け言語なのかと思います。GroovyやScalaの型推論は複雑すぎて、デバッグがつらすぎです。

Nitrous.IOのようにブラウザだけでプログラムとプレビューができる開発環境が気に入っているので、`Visual Studio Online "Monaco"`にも期待してしまいます。

Nitrous.IOでも`parts install dart`できるようで、こっちの方を先に、亜酸化窒素を買わないと。
