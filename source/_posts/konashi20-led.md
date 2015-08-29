title: "konashi 2.0 入門 - Part2: Lチカ"
date: 2015-08-23 20:53:15
categories:
 - IoT
tags:
 - konashi
 - JavaScript
 - jsdoit
 - Lチカ
description: 前回konashi 2.0の概要を理解しました。jsdo.itのコミュニティで共有されているコードを使いお約束のLチカをしてみます。最初なのでデジタルI/0端子は使わずにオンボード出力端子のLEDを使います。
---

[前回](/2015/08/22/konashi20-koshian/)konashi 2.0の概要を理解しました。[jsdo.it](http://jsdo.it/)のコミュニティで共有されているコードを使いお約束のLチカをしてみます。最初なのでデジタルI/0端子は使わずにオンボード出力端子のLEDを使います。

<!-- more -->

## konashi 2.0

konashi 2.0の電源を入れます。コイン型バッテリが付属していますが今回はmicroUSBから電源供給します。

## jsdo.itでFork

konashiは[jsdo.it](http://jsdo.it/)だけでなく、インターネット上に公開すればDropboxに置いたJavaScriptも実行できます。まだ自分でコードを書いていないためコミュニティのコードを使わせていただきます。

最初にWebブラウザから[jsdo.it](http://jsdo.it/)にアカウントを作成してログインします。[まずはLチカ](http://jsdo.it/monakaz/nOMl)のリポジトリを開いてForkします。Finish Editingボタンをクリックして編集画面を終了します。

![led-blinking.png](/2015/08/23/konashi20-led/led-blinking.png)

## konashi.js

手元にあるiPhone 5S、iOS 8.4.1に[konashi.js](https://itunes.apple.com/jp/app/konashi.js-javascript-html/id652940096)をApple Storeからアプリをインストールします。iPhoneとkonashiはBLEで通信するためiPhoneの設定からBluetoothをオンにします。

![konashi.png](/2015/08/23/konashi20-led/konashi.png)

konashi.jsのアプリを起動すると、jsdo.itのコードの検索画面が開き、さきほどForkしたリポジトリが表示されます。タップして選択します。

![jsdoit-search.png](/2015/08/23/konashi20-led/jsdoit-search.png)

リポジトリ画面の左側、プレビューをタップします。

![forked.png](/2015/08/23/konashi20-led/forked.png)

画面のガイドに従って「まわりにあるkonashiを探す」をタップします。

![search.png](/2015/08/23/konashi20-led/search.png)

一覧に表示されたkonashiをタップします。

![konashi2.png](/2015/08/23/konashi20-led/konashi2.png)

konashi.jsアプリが起動します。この画面が表示されていないとコードは実行されません。

![blinking.png](/2015/08/23/konashi20-led/blinking.png)

konashiのLED2が赤く点滅します。

![konashi-led.png](/2015/08/23/konashi20-led/konashi-led.png)
