title: "LG G3 Burgundy Red - Part1: Lollipopアップデート、LTE設定など"
date: 2015-01-21 20:40:52
tags:
 - Android
 - AndroidWear
 - Expansys
 - LGG3
 - Moto360
 - OCN
description: OCN モバイル ONEの音声対応SIMが届きました。キャンペーン中なので追加手数料が無料です。SIMフリー端末はExpansysからLG G3 D855 (Unlocked LTE, 32GB, Burgundy Red)を買いました。さっそくAndroid 5.0 Lollipopにアップデートして、LTEも使えるように設定して使っています。
---

OCN モバイル ONEの音声対応SIMが届きました。キャンペーン中なので追加手数料が無料です。SIMフリー端末はExpansysから[LG G3 D855 (Unlocked LTE, 32GB, Burgundy Red)](http://www.lg.com/global/g3/index.html#gallery_3dvr)を買いました。さっそくAndroid 5.0 Lollipopにアップデートして、LTEも使えるように設定して使っています。

<!-- more -->

## Expansysで購入

合計で54,095円(本体50,395 + 送料1,600 + 関税2,100)でした。香港からクロネコさんが注文してから3日で届けてくれます。いつも速く届けてくれてありがとうございます。

## Lollipopの日本語フォント

開封時はAndroid 4.4.2 KitKatがインストールされています。電源入れて設定をしていると中華フォントが気になります。rootを取ればフォントを変更できますがちょっと悩みます。しばらくするとLollipop配布のメッセージが表示されたのでアップデートしたところ、きれいな日本語フォントになりました。

## LollipopのChrome

LollipopからChromeのタブがアプリ履歴に表示されるようになりました。ただ問題があり、私の環境だとリンクを新しいタブで開こうとすると、かなりの確率でChromeがクラッシュします。復旧するには起動しているアプリを全て消去する必要があり困ります。従来のアプリ内でのタブ切り換えに戻すこともできるので、設定で「タブとアプリを統合」をOFFにしています。

## LTEを有効にする

[こちらの記事](http://juggly.cn/archives/121553.html)にもありますが、LG G3は国際モデルのLTE対応版であっても優先ネットワークにLTEが表示されません。せっかくのLTEもこのままでは選択できず、GSM/WCDMA(自動)がデフォルトです。私の場合は先ほどの記事の通りに電話アプリに番号を入力しても、「テスト画面」の「携帯電話情報」メニューは「アプリはこの携帯電話では作動しません。」とメッセージが表示されて使えません。

いろいろ調べると[こちらに](http://forum.xda-developers.com/lg-g3/general/mod-root-activate-lte-g3-t2819387)T-MobileやAT&Tの場合の対処方が見つかりました。`International mode`の手順通りに電話アプリで`3845#*855#`の番号をダイアルします。その後以下の手順で`LTE / WCDMA`を選択して終了します。

* LTE-Only > Modem Settings > RAT Selection > LTE / WCDMA

これでLTEが使えるようになりました。あと若干問題が残り、Androidを再起動した後は設定が消えてしまうのでこの手順をやり直す必要があります。

## Moto 360に接続がうまくいかない

Zenfone 6のときは良かったのですがMoto 360との接続がうまくいきません。Bluetoothのペアリングまではできますが、その後の接続で止まってしまいます。Android Wearは5.0.1の最新になっているのですが。仕方がないのでAndroid Wear設定から「端末をリセット」するとつながるようになりました。

## まとめ

Isai LGL24の白ロムの方が2万円ほど安くかなり悩みましたが、LG G3を選んでとても満足しています。

* Lollipop
* 背面電源ボタン
* 側面に突起がない
* Burgundy Red
* キャリアの余計なアプリがない


