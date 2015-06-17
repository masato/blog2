title: intel-edison-breakout-board-kit
date: 2015-03-04 00:08:42
tags:
---


スイッチサイエンスで[ntel Edison Kit for Arduino](https://www.switch-science.com/catalog/1958/)が売り切れだったので、[Intel Edison Breakout Board Kit](https://www.switch-science.com/catalog/1957/)を購入しました。最初はLinuxサーバーとして使うのでArduinoは要らないと思っていたのですが、[Grove Starter Kit for Arduino](http://www.seeedstudio.com/depot/Grove-Starter-Kit-for-Arduino-p-1855.html)を見ていたらArduinoとつなぎたくなります。[Intel Edison Board for Arduino](http://ark.intel.com/ja/products/84574/Intel-Edison-Board-for-Arduino)が単体でも売っているようなのでどこかで入手しようと思います。

<!-- more -->

## Ubuntu 14.04でセットアップ

Chromebookから`Ctrl + Alt + t`をタイプしてcroshを起動します。Xfceのデスクトップ環境を実行します。

``` bash
crosh> shell
chronos@localhost / $ sudo startxfce4
```

echo 1 | sudo tee /proc/sys/net/ipv4/ip_forward

Intel EdisonをUSBケーブルで接続するとXfceのデスクトップからマウントが確認できます。

ディスクユーティリティを使ってFAT32にフォーマットする。


## Intel Edison Breakout Board KitでLチカ

Arduinoシールドを使ったLチカはわかりやすいのですが、Breakout Boardを使ってどうすればいいのか

