title: "EspruinoのQuick Startでボタンを押してLチカする"
date: 2015-03-18 22:04:01
tags:
 - Espruino
 - Nodejs
 - SeeedStudio
 - Tessel
 - Blockly
 - Lチカ
description: 2週間前にSeeed Studioから購入したEspruinoがようやく届きました。50ドル以上購入特典で送料無料すると到着まで時間がかかるようです。EspruinoはJavaScript for Microcontrollersこと、JavaScript/Node.jsで操作できるマイコンボードです。また、ChromeアプリのWeb IDEが開発環境として用意されているのも魅力です。Web IDEを使ってQuick Startを読みながらLチカまでやってみます。
---

2週間前に[Seeed Studioから購入](http://www.seeedstudio.com/depot/Espruino-Board-v14-p-2200.html)した[Espruino](http://www.espruino.com/)がようやく届きました。50ドル以上購入特典で送料無料すると到着まで時間がかかるようです。Espruinoは`JavaScript for Microcontrollers`こと、JavaScript/Node.jsで操作できるマイコンボードです。また、ChromeアプリのWeb IDEが開発環境として用意されているのも魅力です。Web IDEを使って[Quick Start](http://www.espruino.com/Quick+Start)を読みながらLチカまでやってみます。

<!-- more -->

## TesselやIntel Edisonと比較

Node.jsでマイクロボードの操作ができる[Tessel](https://tessel.io/)とよく比較されます。Tesselには4つあるポートに挿して使える[モジュール](https://tessel.io/modules)が用意されています。npmでモジュール用のパッケージをインストールして簡単に使えるのが魅力ですが割高になります。一方のEspruinoは安価に購入できて、Arduinoと同じようなセンサーやボードの操作がNode.jsで書けることが魅力です。一般的なセンサー用の[モジュール(ライブラリ)](http://www.espruino.com/Modules)も豊富にあります。
最近はIntel Edisonが便利すぎるのですが、EdisonのYoctoでもNode.jsとNPMが標準で使えます。EdisonでArduino的なことをしていると、Lunixを使いたいのかArduinoを使いたいのかよくわからなくなります。

## USBケーブルで接続する

Windows 7をホストマシンにする場合はEspruinoとUSBケーブルで接続してしばらく待つとOS付属のUSB CDCドライバが自動でインストールされます。OSXやChromebookの場合はドライバはインストール不要でそのまま使えました。

## Espruino Web IDE

Chromeアプリの[Espruino Web IDE](https://chrome.google.com/webstore/detail/espruino-web-ide/bleoifhkdalbjfbobjackfdifdneehpo)をChromeウェブストアからインストールします。アプリランチャーからEspruino Web IDEを起動して左上の`Connect/Disconect`ボタンをクリックすると以下のような接続するポートを選択するダイアログが表示されます。ホストマシンによって異なります。Windowsの場合COM9をクリックするとEspruinoと接続します。

* Windows 7: COM#(COM9など)
* OSX: `/dev/tty.usbmodem1421`
* Chromebook: `/dev/ttyACM0`


![connected.png](/2015/03/18/espruino-quick-start/connected.png)


## ソフトウェアの更新

最初にWeb IDEを起動すると右上に黄色いアラートマークが出ています。マウスオーバーするとあたらしいファームウェアの1v75に更新できるようです。アラートマークをックリックして`FLASHER`画面に移動します。そのまま`FLASH FIRMWARE`ボタンを押すとファームウエア更新に失敗してしまいます。ガイドに従い、EspruinoのRSTボタンとBTN1ボタンを押してbootloaderモードにしてから再度実行します。


## コンソール

Web IDEの左パネルのコンソールはChrome Developper Toolのコンソールのように、直接Node.jsの式を評価できるインタプリタとしても使えます。`1+2`など式を評価後に戻り値がある場合は`=3`など出力します。右パネルのエディタに書いたコードをEspruinoに送って実行するときにはここに標準出力されます。

``` js
>1+2
=3
```

[digitalWrite](http://www.espruino.com/Reference#l__global_digitalWrite)はLEDを操作する関数です。戻り値がないので`=undefined`と表示されます。以下のコードはピン番号に`LED1`、値に`1(true)`を指定してLED1を赤く点灯させます。

``` js
>digitalWrite(LED1,1)
=undefined
```

LEDを消灯する場合は、値に`0(false)`を指定します。

``` js
>digitalWrite(LED1,0)
=undefined
```

## Editor

右パネルにはエディタが表示されます。中央メニューパネルの一番下にあるボタンでNode.jsのCode Editorと[Google Blockly](https://github.com/google/blockly)のGraphical Designerを切り換えることができます。ビジュアルプログラミングが苦手な人も入りやすいです。

### Code Editor

[Espruino 'Quick Reference Card'](http://forum.espruino.com/conversations/264560/)に丁度良いサンプルがあったので試してみます。WEB IDEの右パネルのエディタにコードをコピーします。デバッグ行を少し追加しました。ここからはWindowsからOSXにインストールしたWeb IDEに替えて作業してみます。デフォルトだと`Auto Save`が有効になっているのでコードはGoogleアカウントのCloud Storageに自動保存されます。

```js refcard.js
console.log('start');

// Light LED1
digitalWrite(LED1, 1);
console.log('digitalWrite(LED1,true');

// Blink LED2 for 150ms
digitalPulse(LED2, 1 /*polarity */, 150);

// Turn LED1 off after 1 sec
setTimeout(function() {
  digitalWrite(LED1, 0);
  console.log('digitalWrite(LED1,false');
}, 1000 /* millisecs */);

// 40% duty cycle, 300Hz square wave
analogWrite(A8, 0.4, {freq:300});

// Internal pullup, read value
pinMode(B15, "input_pullup");
console.log('digitalRead(B15): '+digitalRead(B15));

// Read analog value every 100ms, light LED1
setInterval(function() {
  var a = analogRead(A5);
  digitalWrite(LED1, a>0.5);
}, 100);

// When button is pressed
setWatch(function(e) {
  console.log("Press at "+e.time);
}, BTN, { repeat: true, edge: "rising", debounce: 50 });

console.log('end');
```

コードを書き終えたら中央メニューパネルから`Send to Espruino`ボタンをクリックしてEspurinoにアップロードします。

![code-editor.png](/2015/03/18/espruino-quick-start/code-editor.png)


左パネルのコンソールに標準出力されました。EspruinoのBTNを押したイベントも動いています。

![espruino-run.png](/2015/03/18/espruino-quick-start/espruino-run.png)


また、今回はありませんがコードの中でモジュールを`require`していると自動的にモジュールをEspruinoにインストールしてくれるようです。デフォルトでは有効になっていませんが、BETA版としてEspruinoのレジストリにないモジュールはNPMからロードしてくれるオプションもあります。



### Graphical Designer

中央メニューパネルの一番下の`Switch between Code and Graphical Designer`ボタンを押すとエディタをグラフィカルデザイナーに切り換えることができます。JavaScriptで書いたコードが[Blockly](https://github.com/google/blockly)に自動的に変換されるかと思っていましたが、まだ対応していないようです。デフォルトのサンプルからwaitを3秒に変更しました。`Send to Espruino`ボタンをクリックしてEspruinoにアップロードします。ボードのBTN1を押すとLED1が3秒間点灯します。

![blockly-editor.png](/2015/03/18/espruino-quick-start/blockly-editor.png)



