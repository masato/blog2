title: "IDCFクラウドでLinuxMint17 MATEをxrdpから使う - Part1: GUIで開発したい"
date: 2014-06-01 22:18:45
tags:
 - IDCFクラウド
 - LinuxMint17
 - xrdp
 - DartEditor
 - Ubuntu
 - MATE
 - Spark
description: 普段はEmacsがあればよいのでIDEは特に必要なく、iTermやPuttyからSSH接続できる環境をクラウド上に構築して使っています。でも、たまにEclipseベースで開発をする必要があったり、Chromium Browserを使いたいときにGUI環境も必要になります。ちょうど2014-05-31にLinuxMint17 MATEがリリースされたので、IDCFクラウド上に構築しているLinuxMint16環境を新しくしようと思います。New features in Linux Mint 17 MATEを見ながら新機能を確認しているところです。
---

* `Update 2014-08-23`: [LinuxMint17 に Docker開発環境をインストールする](/2014/08/23/linuxmint17-dockerdevenv/)

普段はEmacsがあればよいのでIDEは特に必要なく、iTermやPuttyからSSH接続できる環境をクラウド上に構築して使っています。

でも、たまにEclipseベースで開発をする必要があったり、`Chromium Browser`を使いたいときにGUI環境も必要になります。
ちょうど2014-05-31にLinuxMint17 MATEが[リリース](http://blog.linuxmint.com/?p=2627)されたので、IDCFクラウド上に構築しているLinuxMint16環境を新しくしようと思います。
[New features in Linux Mint 17 MATE](http://www.linuxmint.com/rel_qiana_mate_whatsnew.php)を見ながら新機能を確認しているところです。

<!-- more -->

## LinuxMint16 MATE
普段はLinuxMint16 MATEをIDCFクラウド上に構築してxrdpと一緒に使っています。
Windowsからは標準のリモートデスクトップ、OSXからは[Microsoft Remote Desktop](https://itunes.apple.com/jp/app/microsoft-remote-desktop/id715768417)が使えるので、リモートデスクトップ環境はとても便利です。

## Spark Dart IDE
いつかはcroutonを使わずにChromebookだけで開発したいので、Googleが開発している[Spark](https://github.com/dart-lang/spark)が気になっています。
SparkはDartで書かれたChromeAppベースのIDEです。ウィジェットは`Web Components`をpolyfillしてくれる[Polymer](http://www.polymer-project.org/)というライブラリで書かれています。

Chromebookを使っていると、日常的にChromeAppを使うことになるのですが、もうChromeだけでいいのでは？といつも思います。
ChromeAppのNaClとDartVMの関係はどうなの？とかまだ理解できないことが多いのですが、
きっとこの方向性の先に未来があるような気がしていて、すこしずつ追いついていきたいです。

とりあえず、SparkのDart IDEはどうやってビルドするの？からなのですが、はじめたいと思います。

## Dart Editor
クライアントサイドのAngularDartの勉強をはじめるのには、Emacsだけでも出来そうですが、`Dart Editor`を使った方が捗りそうです。
サーバーサードのDartも、1.3で[パフォーマンスが向上](http://news.dartlang.org/2014/04/dart-13-dramatically-improves-server.html)しています。サーバーサイドWebアプリについても、DartとGoの関係はどうなの？と疑問がつきません。

## まとめ
今のところ、開発用のメインワークステーションとして使っている`LinuxMint16 MATE and xrdp`環境に特に不満はないのですが、
Ubuntu13.10がベースになっているので、2014年7月でサポートが切れてしまいます。
LinuxMint17は`Ubuntu14.04 LTS`がベースなので、2019年4月までサポートされます。
せっかくなので新しいLinuxMint17と[MATE](http://mate-desktop.org/)をセットアップして、`Dart Editor`を使えるようにしていきます。

