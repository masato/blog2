title: 'プログラミング用フォント Ricty Diminished'
date: 2014-06-18 00:46:15
tags:
 - font
 - DartEditor
 - LinuxMint17
 - IDCFクラウド
 - xrdp
description: ブログ用のフォントなども、いつもフォントは悩みます。IDCFクラウド上のLinuxMint17にDartEditorをインストールして開発しているのでですが、xrdpでつないで小さな画面でコードを書いているので、フォントやサイズを変更して使いやすくしたいと思います。ChromebookにUbuntuをインストールしたときは、TakaoPゴシックでしたが、プログラミング用のフォントで有名なRictyもよく使います。Rictyはmonofontなので、日本語をコメントや文字列に入れてもきれいに表示できます。ただIPAのグリフを使っているため、配布ができず、自分でRictyとMigu 1Mを合成するのが面倒なので、新しい環境に入れていませんでしたが、ダウンロードするだけで使えるRicty Diminishedを見つけたので使ってみました。
---

[ブログ用のフォント](/2014/05/07/font-for-blog/)なども、いつもフォントは悩みます。[IDCFクラウド上のLinuxMint17](/2014/06/02/idcf-linuxmint17-part2/)にDartEditorをインストールして開発しているのでですが、xrdpでつないで小さな画面でコードを書いているので、フォントやサイズを変更して使いやすくしたいと思います。

[ChromebookにUbuntuをインストール](/2014/05/02/chromebook-ubuntu/)したときは、TakaoPゴシックでしたが、
プログラミング用のフォントで有名なRictyもよく使います。

Rictyはmonospaceなので、日本語をコメントや文字列に入れてもきれいに表示できます。自分でRictyと`Migu 1M`を合成するのが面倒なので新しい環境に入れていませんでしたが、ダウンロードするだけで使える[Ricty Diminished](http://save.sys.t.u-tokyo.ac.jp/~yusa/fonts/rictydiminished.html)を見つけたので使ってみました。

<!-- more -->

### インストール

フォントをダウンロードします。
``` bash
$ cd ~/Downloads
$ wget http://save.sys.t.u-tokyo.ac.jp/~yusa/fonts/archives/RictyDiminished-3.2.3.tar.gz
```

ホームディレクトリに.fontsディレクトリを作成して、インストールします。
``` bash
$ mkdir ~/.fonts
$ tar zxvf RictyDiminished-3.2.3.tar.gz -C ~/.fonts
RictyDiminished-Bold.ttf
RictyDiminished-BoldOblique.ttf
RictyDiminished-Oblique.ttf
RictyDiminished-Regular.ttf
RictyDiminishedDiscord-Bold.ttf
RictyDiminishedDiscord-BoldOblique.ttf
RictyDiminishedDiscord-Oblique.ttf
```

これで、インストール終了です。かんたん。

### MATEでの設定

LinuxMint17はMATEのデスクトップを使っているので、フォントの設定は以下で行います。
```
Menu -> 設定 -> 外観の設定> フォント 
```


* アプリケーション: TakaoExゴシック: 10
* ドキュメント: TakaoExゴシック: 12
* デスクトップ: TakaoExゴシック: 12
* ウィンドウのタイトル: TakaoExゴシック Bold: 10
* 固定幅のフォント: Ricty Diminished: 12

### Terminalの設定

1.5の倍数で設定するときれいに表示できるようです。
Terminalのフォントサイズはちょっと大きめにします。

```
編集 -> プロファイル -> Default -> 編集 -> Ricty Diminished: 13.5
```

### DartEditorの設定

プログラミングはなるべく大きな文字で、ウィンドウに表示できる範囲の文字数で書きたいです。
```
Tools -> Preferences -> Fonts -> Ricty Dimished :13.5
```

### Chromeの設定

文章を読む場合、Takao Pゴシックの方が読みやすいです。
```
設定 -> ウェブコンテンツ -> フォントをカスタマイズ
```

* 標準フォント: Takao Pゴシック (大)
* Sefifフォント: Takao P明朝
* Sans Serifフォント: Takao Pゴシック

### まとめ
クラウド開発環境にはOSXのiTermからSSHしているため、EmacsもOSXのきれいなフォントでコーディングできます。
WindowsやLinuxMintからもEmacsやエディターを開く場合にも、同様に快適に開発したいです。
