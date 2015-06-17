title: "Spark CoreをOSXとUSB-Serial接続してWi-Fiのセットアップする"
date: 2015-03-01 09:45:29
tags:
 - SparkCore
 - SparkCloud
 - Lチカ
description: Spark CoreをSpark Cloudと接続してAndroidアプリからLチカを試してみます。Spark CoreのセットアップはAndroidアプリを使わずに、OSXとUSB-Serial接続して行います。なぜかAndroidアプリからWi-Fiへ接続できない時にUSB-Serial接続が必要になります。
---

Spark CoreをSpark Cloudと接続してAndroidアプリからLチカを試してみます。Spark CoreのセットアップはAndroidアプリを使わずに、OSXとUSB-Serial接続して行います。なぜかAndroidアプリからWi-Fiへ接続できない時にUSB-Serial接続が必要になります。

<!-- more -->

## OSXとUSB-Serial接続する

AndroidなどのSpark CoreアプリからWi-Fi接続できない状態になった場合など、ホストマシンとSpark CoreをUSB-Serial接続をしてWi-Fiの設定をやり直す場合の手順です。以下のコミュニティサイトを参考にします。

* [App doesn’t connect, USB connection refused - SOLVED](https://community.spark.io/t/app-doesnt-connect-usb-connection-refused-solved/881)

### LEDが青色点滅状態であること

LEDが青色に点滅していて、Wi-Fi接続が待機中であることを確認します。シアンがゆっくり点滅している場合は、Wi-Fi接続中なので、MODEボタンを10秒押してSparkCoreメモリ上のWi-Fi接続情報をクリアします。

今回はホストマシンにMacBook Proを使います。Spark CoreをMacBook Proの右側のUSBポートにケーブルで接続します。lsでデバイスファイルを確認します。

### screenコマンドでシリアル接続

``` bash
$ ls /dev/tty.usb*
/dev/tty.usbmodem1421
```
左のUSBの場合は`tty.usbmodem1411`になります。

``` bahs
/dev/tty.usbmodem1411
```

ターミナルを開きscreenコマンドでSparkCoreとシリアル接続します。

``` bash
$ screen /dev/tty.usbmodem1421 9600
```

`i`キーを押すと、すでにSpark CloudにCoreが登録されている場合はIDが表示されます。

``` bash
Your core id is xxx
```

`w`キーを押すと、Wi-Fiの接続情報を入力できるようになります。今回の環境だと暗号化方式はWPA2です。

``` bash
SSID: xxx
Security 0=unsecured, 1=WEP, 2=WPA, 3=WPA2: 3
Password: xxx
Thanks! Wait about 7 seconds while I save those credentials...

Awesome. Now we'll connect!

If you see a pulsing cyan light, your Spark Core
has connected to the Cloud and is ready to go!

If your LED flashes red or you encounter any other problems,
visit https://www.spark.io/support to debug.

    Spark <3 you!
```

### Spark Cloudへの接続確認

LEDは緑が点滅してWi-Fiに接続します。シアンがすやすや点滅し始めるとCoreのSpark Cloudへの接続が完了した状態です。

OSXのscreenコマンドは、`~/.screenrc`でエスケープキーを`t`にしています。

``` bash
$ cat ~/.screenrc
escape ^Tt
```

`control + t + k`を押して、screenのウインドウを閉じます。

``` bash
Really kill this window [y/n]
```

## Tinkerアプリ

Androidの[Tnkerアプリ](https://play.google.com/store/apps/details?id=io.spark.core.android&hl=ja)と、Spark Coreにデフォルトでインストールされている[Tinkerファームウェア](http://docs.spark.io/tinker/#tinkering-with-tinker-the-tinker-firmware)を使います。


### Tinker設定のクリア

以前設定したTinkerの設定をクリアする場合はメニューから`Clear Tinker`を選択します。

### Tinker

Spark Coreアプリを起動してログインすると、Spark Cloudに登録したCoreが表示されます。

![d7-init.png](/2015/03/01/spark-core-led-blinking/d7-init.png)

右上のD7をタップした後、digitalWriteをタップします。

![d7-setting.png](/2015/03/01/spark-core-led-blinking/d7-setting.png)


右上のD7が赤くなりました。

![d7-red.png](/2015/03/01/spark-core-led-blinking/d7-red.png)

D7をタップしてHIGHにすると、Spark CoreのUSBポート右側のLEDが青く点灯します。もう一度D7をタップしてLOWにすると点灯が終了しす。

![d7-high.png](/2015/03/01/spark-core-led-blinking/d7-high.png)

