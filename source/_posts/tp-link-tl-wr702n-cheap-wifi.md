title: "TP-LINKのWR702NモバイルルーターのClientモードをOSXから設定する"
date: 2015-04-17 12:46:28
categories:
 - IoT
tags:
 - TP-LINK
 - Wi-Fi
 - WR702N
 - モバイルルーター
description: TP-LINKのWR702Nは安価で機能が豊富なモバイルルーターです。接続モードがAP、Router、Repeater、Bridge、Clientと用意されています。特にClientモードは無線LANカードとして使えるのでArduioとEthernetケーブルで接続するとArduinoの安価なWi-Fi環境が作れます。ただしブラウザから使える設定画面が中国語です。NodeMCUもそうですがIoTは中国語も読めないと困ることが多くなりそうです。日本語マニュアルが付属していますがWindows用なので参考にしながらOSXで設定をしていきます。
---

TP-LINKの[WR702N](http://www.amazon.co.jp/dp/B005NEU3WS/)は安価で機能が豊富なモバイルルーターです。接続モードがAP、Router、Repeater、Bridge、Clientと用意されています。特にClientモードは無線LANカードとして使えるのでArduioとEthernetケーブルで接続するとArduinoの安価なWi-Fi環境が作れます。ただしブラウザから使える設定画面が中国語です。[NodeMCU](http://www.nodemcu.com/index_cn.html)もそうですがIoTは中国語も読めないと困ることが多くなりそうです。日本語マニュアルが付属していますがWindows用なので参考にしながらOSXで設定をしていきます。

<!-- more -->

## OSXとEthernetケーブルで接続する

モバイルルーターの設定はホストマシンをEthernetケーブルで接続してブラウザから行います。MacBook Proを使っていますがEthernetポートがありません。[USB Ethernet LANアダプター](http://store.shopping.yahoo.co.jp/taobaonotatsujinpro/usbethernet1.html)を購入してモバイルルーターと接続します。

システム環境設定 > ネットワーク > USB Ethernet LANアダプターの設定を開きます。

![static-usb-lan.png](/2015/04/17/tp-link-tl-wr702n-cheap-wifi/static-usb-lan.png)

* IPv4の設定: 手入力
* IPアドレス: 192.168.1.x (xは2-252の間の任意)
* ルーター: 192.168.1.253 (モバイルルーターのIPアドレス)


## モバイルルーターの設定

OSXからブラウザを開き`192.168.1.253`に接続してパスワードに`admin`を入力して下に見えるボタンを押してログインします。

![password.png](/2015/04/17/tp-link-tl-wr702n-cheap-wifi/password.png)

### 接続モードをClientにする

メニューから`工作模式`のリンクを開き接続モードを設定します。Clientモードをチェックして右下のボタンをクリックして次に進みます。

![client-menu.png](/2015/04/17/tp-link-tl-wr702n-cheap-wifi/client-menu.png)

画面中央のボタンを押してアクセスポイントを検索します。

![search-ap.png](/2015/04/17/tp-link-tl-wr702n-cheap-wifi/search-ap.png)

見つかったAPの一覧が表示されます。接続したいAPのリンクをクリックします。

![select-ap.png](/2015/04/17/tp-link-tl-wr702n-cheap-wifi/select-ap.png)

アクセスポイントのパスワードを入力して、右下のボタンを押して次に進みます。

![ap-password.png](/2015/04/17/tp-link-tl-wr702n-cheap-wifi/ap-password.png)

設定変更の確認のためブラウザのアラートが表示されます。OKをクリックするとモバイルルーターが再起動して設定が反映されます。

![restart.png](/2015/04/17/tp-link-tl-wr702n-cheap-wifi/restart.png)


## DHCPからIPアドレスを取得する

OSXに戻りネットワーク設定を変更します。モバイルルーターからDHCPでIPアドレスを取得するようにします。IPアドレスがうまく取得できない場合はモバイルルーターの電源を入れ直して再起動すると成功するようです。

システム環境設定 > ネットワーク > USB Ethernet LANアダプターの設定を開きます。

![dhcp.png](/2015/04/17/tp-link-tl-wr702n-cheap-wifi/dhcp.png)


* IPv4の設定: DHCP サーバーを使用