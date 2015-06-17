title: "IDCFクラウドでUbuntu MATE 14.10とxrdpを使う"
date: 2015-01-13 01:54:00
tags:
 - IDCFクラウド
 - Ubuntu
 - MATE
 - xrdp
 - 月額500円
description: Ubuntu MATE 14.10 が2014年10月にリリースされました。Linux Mintで使い慣れている軽量デスクトップのMATEがUbuntuでも正式に使えます。昔懐かしい感じのデスクトップです。月額500円のIDCFクラウド最小構成でインストールします。リモートデスクトップはxrdpを使いWindowsとOSXのどちらからも日本語環境で接続できるようにします。
---

Ubuntu MATE 14.10 が2014年10月にリリースされました。Linux Mintで使い慣れている軽量デスクトップのMATEがUbuntuでも正式に使えます。昔懐かしい感じのデスクトップです。月額500円のIDCFクラウド最小構成でインストールします。リモートデスクトップはxrdpを使いWindowsとOSXのどちらからも日本語環境で接続できるようにします。

<!-- more -->


### ISO作成

[Download](https://ubuntu-mate.org/download/)ページからubuntu-mate-14.10-desktop-amd64.isoのURLをコピーして、IDCFクラウドのISO作成画面で入力します。

* ISO名: ubuntu-mate-14.10-desktop-amd64.iso
* 説明: ubuntu-mate-14.10-desktop-amd64.iso
* URL: http://pub.mate-desktop.org/iso/ubuntu-mate/release/amd64/ubuntu-mate-14.10-desktop-amd64.iso
* ゾーン: tesla
* OSタイプ: Ubuntu 12.04 (64-bit)
* エクスポート: 有効
* ブータブル: 有効

### 仮想マシン作成

月額500円構成なので、仮想マシンのスペックはlight.S1 (200円)、ボリュームは15 GB (300円)を指定します。

* 仮想マシン: light.S1 CPUx1 Mem 1GB
* イメージ:  ubuntu-mate-14.10-desktop-amd64.iso
* ボリューム: 15GB
* 仮想マシン名: ubuntu-mate-1410
* SSH Key: (None)
* プライベートIPアドレス: AUTO
 
### Ubuntuをディスクにインストール

仮想マシンが起動したらWebコンソールを開きます。日本語を選択してからはほぼデフォルトで、ISOから起動したUbuntuをディスクにインストールします。インストールが成功したらISOは仮想マシンからデタッチします。

* Welcome
 * 「日本語」を選択
 * 「Ubuntuをインストール」をクリック
* インストールの種類
 * 「ディスクを削除してUbuntuをインストール」にチェック
 *  「インストール」をクリック
* どこに住んでいますか？
 * Tokyo
* キーボードレイアウト
 * キーボードレイアウトの選択: Japanese / Japanese
* あなたの情報を入力してください
 * あなたの名前: Masato Shimizu
 * コンピュータの名前: ubuntu-mate-1410
 * ユーザー名の入力: mshimizu
 * パスワード: xxx
 * パスワードの確認: xxx
 * 「自動的にログインする」にチェック
* インストールが完了しました
 * 「今すぐ再起動する」をクリック
* wait-for-state stop/waitingの画面が表示される
 * Webポータル画面からインスタンスを選択してISOをデタッチ
 * デタッチ 成功したら、コンソールでENTERを押す

### SSHサーバーのインストール

OSの再起動後Webコンソールを開きます。

{% img center /2015/01/13/idcf-ubuntu-mate-1410-xrdp/ubuntu-mate-1410.png %}

MATE端末を開きSSHサーバーのインストールをします。

* 左上メニュー -> システムツール -> MATE端末

``` bash
$ sudo apt-get update 
$ sudo apt-get install -y openssh-server
```

### vimのインストール

お好みでnanoからvimにエディタを変更します。

``` bash
$ sudo apt-get install vim
$ sudo update-alternatives --set editor /usr/bin/vim.basic
```

### xrdpのインストール

xrdpをインストールします。

``` baxh
$ sudo apt-get install xrdp vnc4server
$ xrdp --version

xrdp: A Remote Desktop Protocol server.
Copyright (C) Jay Sorg 2004-2011
See http://xrdp.sourceforge.net for more information.
Version 0.6.1
```

### キーボードの設定

キーボード \と_を入力できるようにします。/etc/default/keyboardはデフォルトのまま105のレイアウトを使います。

``` bash /etc/default/keyboard
XKBMODEL="pc105"
XKBLAYOUT="jp"
XKBVARIANT=""
XKBOPTIONS=""
```

xrdpで日本語キーボードを認識させるため、キーマップを変更します。

``` bash
$ cd /etc/xrdp
$ sudo wget http://www.mail-archive.com/xrdp-devel@lists.sourceforge.net/msg00263/km-e0010411.ini
$ sudo mv km-e0010411.ini km-0411.ini 
$ sudo chmod 644 km-0411.ini
$ sudo ln -s km-0411.ini km-e0010411.ini
$ sudo ln -s km-0411.ini km-e0200411.ini
$ sudo ln -s km-0411.ini km-e0210411.ini
```

### 日本語入力設定

日本語入力にはibus-mozcを使います。

```
$ sudo apt-get install ibus-mozc
```

### OSXからxrdpで接続する

xrdpをリスタートします。

```
$ sudo service xrdp restart
```

OSXのクライアントは[Microsoft Remote Desktop Connection Client for Mac 2.1.1](http://www.microsoft.com/ja-jp/download/details.aspx?id=18140)をインストールします。App Storeで配布されている[Microsoft Remote Desktop](https://itunes.apple.com/jp/app/microsoft-rimoto-desukutoppu/id714464092?mt=8)からxrdpでUbuntu MATEに接続すると、日本語キーボードが認識されないので\と_が入力できません。

入力モードで日本語-Mozcを選択します。OSXの場合は英数キーでトグルできます。

### ディレクトリ名を英語にする

ホームディレクトリに日本語名のディレクトリがあるので英語表記に変換します。デスクトップにログインしてMATE端末を開きます。

``` bash
$ LANG=C xdg-user-dirs-gtk-update
Moving DESKTOP directory from デスクトップ to Desktop
Moving DOWNLOAD directory from ダウンロード to Downloads
Moving TEMPLATES directory from テンプレート to Templates
Moving PUBLICSHARE directory from 公開 to Public
Moving DOCUMENTS directory from ドキュメント to Documents
Moving MUSIC directory from ミュージック to Music
Moving PICTURES directory from ピクチャ to Pictures
Moving VIDEOS directory from ビデオ to Videos
```

### Chromeのインストール

OSXのRDPのメニューから環境設定画面を開き、表示タブの色を約1670万色にします。デフォルトの約32000色の場合、Chromeがきれいに表示されません。

{% img center /2015/01/13/idcf-ubuntu-mate-1410-xrdp/xrdp-color.png %}

Firefoxを起動して[Chrome](https://www.google.com/chrome/browser/desktop/index.html)のページから64bit .deb (Debian/Ubuntu 版)をチェックしダウンロードします。

依存依存パッケージをインストールします。

``` bash
$ sudo apt-get install libcurl3 libappindicator1
```

Chromeをインストールします。

``` bash
$ cd ~/Downloads
$ sudo dpkg -i google-chrome-stable_current_amd64.deb
```

Chromeもきれいに表示できました。


{% img center /2015/01/13/idcf-ubuntu-mate-1410-xrdp/ubuntu-mate-chrome.png %}





