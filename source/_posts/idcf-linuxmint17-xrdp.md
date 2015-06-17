title: "IDCFクラウドにLinux Mint 17 MATEとxrdpをインストールする"
date: 2014-12-31 16:52:17
tags:
 - IDCFクラウド
 - LinuxMint17
 - MATE
 - xrdp
 - DockerDevEnv
 - Ionic
 - Cordova
 - OnsenUI
 - AndroidSDK
 - HTML5HybridMobileApps
description: Ionicの開発用にWebブラウザからプレビューできる環境をDocker上につくりました。今度はエミュレーターを動かせるデスクトップ環境を用意します。以前Linux Mint 17 MateのデスクトップをIDCFクラウド上に構築しましたが、CordovaとIonicの開発用に新しく仮想マシンを作ることにします。
---

Ionicの開発用にWebブラウザからプレビューできる環境を[Dockerコンテナ上につくりました](2014/12/30/docker-devenv-ionic-cordova/)。今度はエミュレーターを動かせるデスクトップ環境を用意します。以前Linux Mint 17 Mateのデスクトップを[IDCFクラウド上に構築](2014/06/02/idcf-linuxmint17-part2/)しましたが、CordovaとIonicの開発用にも、新しく仮想マシンを作ることにします。

<!-- more -->

### ISOの登録

2014-11-29に[Linux Mint 17.1 Rebecca](http://www.linuxmint.com/download.php)がリリースされていますが、以前インストールに成功しているLinux Mint 17 Qianaを使うことにします。

IDCFクラウドのポータルにログインして、ISOの登録をします。

* ISO名: linuxmint-17-mate-dvd-64bit
* 説明: linuxmint-17-mate-dvd-64bit
* URL:  http://ftp.stust.edu.tw/pub/Linux/LinuxMint/isos//stable/17/linuxmint-17-mate-dvd-64bit.iso
* ゾーン: tesla
* OSタイプ: Ubuntu 12.04 (64-bit)
* エクスポート: 有効
* ブータブル: 有効

### 仮装マシンの作成

登録したISOから仮装マシンを作成します。

* 仮想マシン: standard.M8 ( 2 CPU x 2.4 GHz / 8GB RAM )
* イメージ: mint-17-mate-dvd-64bit
* ボリューム: 80GB
* 仮想マシン名: mint-mate-17
* SSH Key: (None)
* プライベートIPアドレス: AUTO

IPアドレスはDHCPで、10.3.0.131がアサインされました。

### Install Linux Mint

仮装マシン画面からコンソールを起動して、デスクトップ上のInstall Linux Mint のアイコンをダブルクリックします。以下の手順でディスクにインストールしていきます。

* Welcome画面で日本語を選択し、「続ける」ボタン
* Linux Mintのインストール準備、「続ける」ボタン
* どこに住んでいますか？、Tokyo、「続ける」ボタン

* インストールの種類
 * ディスクを削除してLinux Mintをインストール
  * Use LVM with the new Linux Mint instllationにチェック
  * 「インストール」ボタン

* キーボードレイアウト
 * デフォルトのまま
 * キーボードレイアウトの選択: 日本語、日本語
 * 「続ける」ボタン

* ユーザー
 * あなたの名前: Masato Shimizu
 * コンピューターの名前: mint-17
 * ユーザー名: mshimizu
 * パスワードの入力: xxx
 * 自動的にログインするにチェック
 * 「続ける」ボタン


Linux Mintへようこその画面が表示され、インストールが開始します。インストールを完了して「今すぐ再起動する」ボタンを押します。 メッセージが表示が表示されるのでこのままの状態で、ポータル画面から仮装マシンを選択してISOをデタッチします。

> Please remove installation media and close the tray (if any) then press ENTER:

デタッチが成功したら、上記メッセージが表示されているコンソールでENTERを押します。

### コンソールからSSHをインストール

仮装マシンの再起動後、コンソールにWelcome Screenが表示されます。

メニューから端末を開いてSSHをインストールします。

``` bash
$ sudo apt-get update
$ sudo apt-get install openssh-server
```
mint-mate-17の仮装マシンにパスワード認証でSSHでログインします。

``` bash
$  ssh mshimizu@linux-mint-mate-17 -o PreferredAuthentications=password
mshimizu@linux-mint-mate-17's password:
Welcome to Linux Mint 17 Qiana (GNU/Linux 3.13.0-24-generic x86_64)

Welcome to Linux Mint
 * Documentation:  http://www.linuxmint.com
mshimizu@linux-mint-17 ~ $
```

SSH接続が確認できたら、ブラウザのコンソールはもう使わないので閉じます。

### 基本ツールのインストール

nanoは使いにくいのでデフォルトのエディタをvim.basicにします。

``` bash
$ sudo apt-get install vim
$ sudo update-alternatives --set editor /usr/bin/vim.basic
```

Firefoxを起動してChromeを[ダウンロード](https://www.google.com/chrome/browser/desktop/index.html
)します。

64bit .deb (Debian/Ubuntu 版)をインストールします。

``` bash
$ cd ~/Downloads
$ sudo apt-get install libcurl3
$ sudo dpkg -i google-chrome-stable_current_amd64.deb
```

デフォルトのブラウザを確認します。

``` bash
$ sudo update-alternatives --config x-www-browser
alternative x-www-browser (/usr/bin/x-www-browser を提供) には 2 個の選択肢があります。

  選択肢    パス                         優先度  状態
------------------------------------------------------------
* 0            /usr/bin/google-chrome-stable   200       自動モード
  1            /usr/bin/firefox                40        手動モード
  2            /usr/bin/google-chrome-stable   200       手動モード

現在の選択 [*] を保持するには Enter、さもなければ選択肢の番号のキーを押してください:
```

左下のMenuボタンからお気に入りのアプリのパネルを開き、ウェブ・ブラウザでChromeを選択します。

``` bash
Menu -> コントロールセンター -> お気に入りのアプリ -> ウェブ・ブラウザ -> Google Chrome
```

Chromeが起動するか確認します。

``` bash
$ xdg-open http://www,yahoo.co.jp
```


### xrdpのインスト-ル

xrdpをインストールします。

``` bash
$ sudo apt-get update
$ sudo apt-get install xrdp vnc4server
xrdp (0.6.0-1) を設定しています ...
 * Generating xrdp RSA keys......

Generating 512 bit rsa key...

ssl_gen_key_xrdp1 ok

saving to /etc/xrdp/rsakeys.ini

   ...done.
 * Starting Remote Desktop Protocol server
   ...done.
Processing triggers for ureadahead (0.100.0-16) ...
```

xrdpのバージョンを確認します。

``` bash
$ xrdp --version

xrdp: A Remote Desktop Protocol server.
Copyright (C) Jay Sorg 2004-2011
See http://xrdp.sourceforge.net for more information.
Version 0.6.1
```

MATE用のX認証ファイルを作成します。

``` bash
$ touch ~/.Xauthority
```

### 3389ポートを使う

IDCFクラウドのポータル画面から、IPアドレスのファイアウォールにRDPの3389ポートを開放します。

### Windows 7の場合、付属のリモートデスクトップを使う

Windows 7の場合OSに付属しているリモートデスクトップがそのまま使えます。


### OSXの場合、Microsoft Remote Desktop Connection Client for Macを使う

OSXで使えるRDPクライアントは2種類ありますが、[Microsoft Remote Desktop Connection Client for Mac 2.1](http://www.microsoft.com/ja-jp/download/details.aspx?id=18140)をダウンロードしてインストールします。インストーラーを右クリックして開くを実行します。App Storeで配布されている[Microsoft Remote Desktop](https://itunes.apple.com/jp/app/microsoft-remote-desktop/id715768417)を使うと、日本語キーボードが正しく認識されず、\と_が入力できないので使い物になりません。

RDPのメニューから環境設定画面を開き、表示タブの色を約1670万色にします。デフォルトの約32000色の場合、Chromeがきれいに表示されません。

{% img center /2014/12/31/idcf-linuxmint17-xrdp/xrdp-color.png %}


### キーボード \と_を入力できるようにする

`/etc/default/keyboard`はデフォルトのまま使います。

``` bash /etc/default/keyboard
XKBMODEL="pc105"
XKBLAYOUT="jp"
XKBVARIANT=""
XKBOPTIONS=""
```

[Ubuntu 12.04でxrdpで日本語キーボードを認識させる](https://gist.github.com/devlights/6337527)を参考にして、キーマップを変更します。

``` bash
$ cd /etc/xrdp
$ sudo wget http://www.mail-archive.com/xrdp-devel@lists.sourceforge.net/msg00263/km-e0010411.ini
$ sudo mv km-e0010411.ini km-0411.ini 
$ sudo chmod 644 km-0411.ini
$ sudo ln -s km-0411.ini km-e0010411.ini
$ sudo ln -s km-0411.ini km-e0200411.ini
$ sudo ln -s km-0411.ini km-e0210411.ini
```

xrdpを再起動します。

``` bash
$ sudo service xrdp restart
```

### 日本語入力にibus-mozcを使う

日本語入力にibus-mozcをインストールします。

``` bash
$ sudo apt-get install ibus-mozc
```

xrdpからログアウトして再接続します。入力モードで「あ]が表示されます。Windows7の場合、半角キーで日本語と半角英数のトグルができます。OSXの場合は英数キーでトグルできます。

### ディレクトリ名を英語にする

日本語環境にするとホームディレクトリの名前も日本語になります。デスクトップにログインして端末を開きます。

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
