title: "LinuxMint17とxrdpでAngularDartの勉強 - Part1: Dart"
date: 2014-06-05 01:13:50
tags:
 - LinuxMint17
 - AngularDart
 - Dart
 - DartEditor
description: AngularDartを始める場合、オフィシャルのAngularDart Tutorialを読み進めていくのがよさそうです。他の情報だとAPIが古かったり、絶賛開発中なので、また仕様が大きく変わる可能性があります。まずはDartの勉強からですが、オフィシャルのチュートリアルやドキュメントの質が他の言語に比べてとても高いです。新しい言語を広めるために、開発者への配慮と、Googleの本気度がうかがえます。
---

[AngularDart](https://angulardart.org/)を始める場合、オフィシャルの[AngularDart Tutorial](https://angulardart.org/tutorial/)を読み進めていくのがよさそうです。他の情報だとAPIが古かったり、絶賛開発中なので、また仕様が大きく変わる可能性があります。
まずはDartの勉強からですが、オフィシャルのチュートリアルやドキュメントの質が他の言語に比べてとても高いです。
新しい言語を広めるために、開発者への配慮と、Googleの本気度がうかがえます。

<!-- more -->

### xrdp環境
xrdpでOSXやWindowsから便利に接続していますが、セッションがうまく切れないようです。
`Dart Editor`を開いたままセッションを閉じるとプロセスが残ります。昨日はこれが原因でChromeが起動しない状態になりました。
気づくまで、Dartiumを疑っていました。

### アップデートマネージャ
画面下パネルに頻繁にアップデートの通知が出ますが、画面からはアップデートができません。
チェックマークが表示されていないと気になるので、コンソールを開き`sudo apt-get upgrade`しています。

### はじめてのDart
まだ始めたばかりですが、AngularDartというかDart言語自体がすごく好きになりました。以下のリンクを順番に進めていきます。

* [Before You Begin](https://angulardart.org/tutorial/01-before-you-begin.html)
* [Avast, Ye Pirates: Write a Web App](https://www.dartlang.org/codelabs/darrrt/)
* [A Tour of the Dart Language](https://www.dartlang.org/docs/dart-up-and-running/contents/ch02.html)
* [The Dart Tutorials](https://www.dartlang.org/docs/tutorials/)


### Dart書籍
日本語でもManning翻訳本が出ました。Kindle版の`What is Dart?`は0円です。
Packtは2014年の出版です。私はいつもManningの`in Action`を最初に読むようにしています。

* [Dart in Action](http://manning.com/buckett/)
* [Learning Dart](http://www.packtpub.com/learning-dart/book)
* [What is Dart?](http://www.amazon.co.jp/dp/B007K0T824/)
* [プログラミング言語Dart](http://www.amazon.co.jp/dp/B00JB3CWZI)
* [Dart: Up and Running](http://www.amazon.co.jp/dp/1449330894)
* [Dart for Hipsters](http://dart4hipsters.com/)
