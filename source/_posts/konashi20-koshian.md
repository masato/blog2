title: "konashi 2.0 入門 - Part1: konashiとKoshian"
date: 2015-08-22 16:05:32
categories:
 - IoT
tags:
 - フィジカル・コンピューティング
 - konashi
 - BLE
 - Koshian
 - JavaScript
 - jsdoit
description: ユカイ工学株式会社が開発しているフィジカル・コンピューティング・ツールキット、konashiのメジャーアップデート版であるkonashi 2.0を購入しました。フロントエンドエンジニアやデザイナー、アーティストでも一般的になっているWebベースのJavaScriptでスマホからハードウェアを制御するプロトタイプが簡単に作ることができます。最初にkonashi、konashi 2.0、Koshian、konashi.jsの関係がちょっとわかりづらく混乱してしまうので、整理して概念と構成について調べてみます。
---

[ユカイ工学株式会社](http://www.ux-xu.com/)が開発しているフィジカル・コンピューティング・ツールキット、konashiのメジャーアップデート版である[konashi 2.0](http://www.ux-xu.com/product/konashi2-0)を購入しました。フロントエンドエンジニアやデザイナー、アーティストでも一般的になっているWebベースのJavaScriptでスマホからハードウェアを制御するプロトタイプが簡単に作ることができます。最初にkonashi、konashi 2.0、[Koshian](http://www.m-pression.com/ja/solutions/boards/koshian)、[konashi.js](http://konashi.ux-xu.com/kjs/)の関係がちょっとわかりづらく混乱してしまうので、整理して概念と構成について調べてみます。

<!-- more -->

## フィジカル・コンピューティング

[フィジカル・コンピューティング](https://en.wikipedia.org/wiki/Physical_computing)とはニューヨーク大学のDan O'Sullivan教授が提案した教育プログラムです。人の行動や生活環境によりそったコンピュータとの意思疎通の方法を模索する研究や活動です。コンピューターにセンサーなどの入出力デバイスをつなぎ、人と情報をやりとりすることで、生活を便利にしたり、新しいユーザー体験を生み出すモノづくりが可能になると考えられています。

ただし、ソフトウェア、ハードウェア、デザイン、アートの各分野の幅広い知識が必要です。専門の教育を受けていないユーザーがプロトタイプを作るのは困難でした。konashiを使うとJavaScriptというWebで一般的な知識で、フロントエンジニアやデザイナーでも自分のアイデアから、簡単にモノづくりのプロトタイピングができるようになります。

## konashi

### 構成

konashiは大きく2つの部品から構成されています。

* CPUを搭載したBluetooth SMARTモジュール
* I/O接続を提供するベースボード

[konashi 2.0](http://www.ux-xu.com/product/konashi2-0)ではBluetooth SMART モジュールが[Koshian](http://www.m-pression.com/ja/solutions/boards/koshian)に変更になりました。半額と低価格になり、通信インタフェースが強化されました。また、KoshianのOTA機能によりkonashi.js(iOSアプリ)からファームウェアの更新も可能になりました。アップデートでSPI通信への対応が予定されています。

### Koshian

[Koshian](http://www.m-pression.com/ja/solutions/boards/koshian)は技適マーク取得済のBluetooth SMART モジュールを搭載しています。[980円](https://store.macnica.co.jp/products/mpression_mp-ksn001b)と、とても安価に購入できます。

Koshianではオリジナルのファームウェアに加えて、一部機能を除くkonashi 2.0互換ファームウェアも使うことができます。このためKoshianモジュールをkonashiのベースボードに組み込まなくても単体でkonashi.js(iOSアプリ)からハードウェアのI/O通信を制御することができます。

Koshianはピンヘッダのピッチが狭かったり、I/Oの引き出しも面倒です。Koshianがベースボードに実装されているkoshian 2.0ではコイン型バッテリやmicroUSBポート、アナログ入力やI2C通信端子が最初から使えるので初心者には便利です。

### koshian.js

`***.js`と書くとNode.jsやEmber.jsのようにJavaScriptのプラットフォームやライブラリに見えますが、konashi.jsはiOSアプリです。この名前のおかげで最初は混乱しました。

アプリはApple Storeから[インストール](https://itunes.apple.com/jp/app/konashi.js-javascript-html/id652940096)します。koshian.jsは[jsdo.it](http://jsdo.it/konashijs/)のWeb上に保存してあるJavaScriptのコードを実行して、konashiを通じてセンサーやデバイスなどハードウエアを制御することができます。

### jsdo.it

[jsdo.it](http://jsdo.it/)は[面白法人カヤック](http://www.kayac.com/)が開発しています。フロントエンドエンジニアのためのJavaScript/HTML5/CSSのリポジトリを共有できるコミュニティサービスです。初心者でもkonashi.jsアプリのブラウザからjsdo.it上で他の人が共有しているコードを実行することができます。自分で動かして真似しながら勉強してフィードバックしていくコミュニティのよい環境ができています。

## konashiの名前の由来

[konashi・Koshian 名前の由来](https://store.macnica.co.jp/library/96029)によると、konashiとは和菓子の材料となる生地の「こなし」から来ています。和菓子職人はこなし作りが「こなせて」一人前だそうで、konashiを基礎にして新しいモノづくりをしていこうという意味が込められているようです。