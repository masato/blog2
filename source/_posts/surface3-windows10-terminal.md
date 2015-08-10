title: "Surface 3に開発環境をつくる - Part2: Minttyのターミナル"
date: 2015-08-01 23:04:52
tags:
 - Surface3
 - Windows10
 - MicrosoftEdge
 - Pocket
 - Mintty
description: Surface3のディスプレイは10.8インチで画面解像度は1920×1280ドットです。デフォルトでは「テキスト、アプリ、その他の項目のサイズを変更する」が150%に設定されています。タブレットモードで使う場合はこれで十分ですが、ノートPCとして使う場合はとても文字が小さく感じます。このままプログラミングで使うと目が疲れてしまうので、フォントを変えたり、OSのテキストサイズを変更する必要があります。コマンドプロンプトは開発環境に適していないので、Minttyをインストールして使います。
---

Surface3のディスプレイは10.8インチで画面解像度は1920×1280ドットです。デフォルトでは「テキスト、アプリ、その他の項目のサイズを変更する」が150%に設定されています。タブレットモードで使う場合はこれで十分ですが、ノートPCとして使う場合はとても文字が小さく感じます。このままプログラミングで使うと目が疲れてしまうので、フォントを変えたり、OSのテキストサイズを変更する必要があります。コマンドプロンプトは開発環境に適していないので、[Mintty](http://mintty.github.io/)をインストールして使います。


<!-- more -->

## OSのテキストサイズ

Windows10では画面を右端から左にスワイプすると「チャーム」の代わりに「アクションセンター」が表示されます。「すべての設定」をタップすると「設定」画面が表示されます。右上の「システム」をタップすると「ディスプレイのカスタマイズ」画面になります。設定は好みですが、プログラミングで使う場合、テキストのサイズは200%の最大にすると見やすいです。

![display.png](/2015/08/01/surface3-windows10-terminal/display.png)

## Mintty

[MinGW](http://www.mingw.org/)から[MSYS](http://www.mingw.org/wiki/msys)をインストールしたあとに`mingw-get`コマンドから[Mintty](http://mintty.github.io/)をインストールします。

### MinGW

mingw-get-setup.exeを[ダウンロード](http://sourceforge.net/projects/mingw/files/Installer/mingw-get-setup.exe/download)して実行します。[MinGW](http://www.mingw.org/)はWindows用のGNUツールです。MinGWからMSYSをインストールしたいのでmsys-base A Basic MSYS Installation (meta)にチェックを入れます。MSYSはWindowsでBashを動かすためのツールです。

![mingw.png](/2015/08/01/surface3-windows10-terminal/mingw.png)


### 環境変数

左下のWindowsボタンからシステムプロパティのダイアログを表示して環境変数ボタンを押します。

* コントロール パネル > システムとセキュリティ > システム > システムの詳細設定 > 環境変数


![system-property.png](/2015/08/01/surface3-windows10-terminal/system-property.png)


ユーザー環境変数に`HOME`を作成します。

```text
C:\MinGW\msys\1.0\home\{username}
```

### MSYS

以下のフォルダにあるmsysを起動します。

```
C:\MinGW\msys\1.0\msys
```

シェルが起動するので、updateをした後にminttyをインストールします。

```
$ mingw-get update
$ mingw-get install mintty
```

### Ricty Diminishedフォント

プログラミングに適したフォントの[Ricty Diminished](https://github.com/yascentur/RictyDiminished)をインストールします。リポジトリのページから[RictyDiminished-master.zip](https://github.com/yascentur/RictyDiminished/archive/master.zip)をダウンロードします。zipを解凍してすべてのフォントをインストールします。

### Minttyの設定

MSYSのbinにインストールされたminttyを起動します。またminntyを右クリックしてタスクバーとスタート画面にピン留めをしておきます。

```
C:\MinGW\msys\1.0\bin\mintty
```

タイトルバーを右クリックして`Options`を選択します。好みですが以下のように設定します。

* Text
 * Font: Ricty Dimished Discord: 16-point
 * Locale: ja_JP
 * Characterset: UTF-8

![text.png](/2015/08/01/surface3-windows10-terminal/text.png)

* Window
 * Columns: 100
 * Rows: 30

![window.png](/2015/08/01/surface3-windows10-terminal/window.png)


## 開発環境

MinttyではエディタとSSHが使えればよいので、vimとopensshをインストールします。

```
$ mincw-get install msys-vim
$ mincw-get install msys-openssh
```

ssh-keygenコマンドでキーペアを作成して、ssh-copy-id 
などを使い、ログインするサーバーに公開鍵を登録しておきます。


