title: "IDCFクラウドでLinuxMint17 MATEをxrdpから使う - Part2: インストール編"
date: 2014-06-02 23:00:29
tags:
 - IDCFクラウド
 - LinuxMint17
 - xrdp
 - DartEditor
 - Ubuntu
 - MATE
 - Spark
description: 前回に続いて、IDCFクラウド上にLinuxMint17 MATEのリモートデスクトップ環境を構築します。いつも日本語入力と日本語キーボードの設定で嵌まるのですが、LinuxMint16のときと同じ手順で大丈夫でした。xrdpを使う前は、FreeNXとNoMachineクライアントで構築していたのですが、NoMachine4になってライセンスや設定方法がよくわからなくなりました。もともと日本語化が面倒だったので、xrdpが今のところ一番便利に使えます。
---

* `Update 2014-12-31`: [IDCFクラウドにLinux Mint 17 MATEとxrdpをインストールする](/2014/12/31/idcf-linuxmint17-xrdp/)
* `Update 2014-08-23`: [LinuxMint17 に Docker開発環境をインストールする](/2014/08/23/linuxmint17-dockerdevenv/)


[Part1](/2014/06/01/idcf-linuxmint17-part1/)に続いて、IDCFクラウド上にLinuxMint17 MATEのリモートデスクトップ環境を構築します。
いつも日本語入力と日本語キーボードの設定で嵌まるのですが、LinuxMint16のときと同じ手順で大丈夫でした。

xrdpを使う前は、FreeNXとNoMachineクライアントで構築していたのですが、NoMachine4になってライセンスや設定方法がよくわからなくなりました。もともと日本語化が面倒だったので、xrdpが今のところ一番便利に使えます。

<!-- more -->

### インストールの準備

最初にインストール用のISOを[ダウンロード](http://linuxmint.com/download.php)します。
`idcf-compute-api`を使う場合はregisterIsoコマンドを使います。IDCFクラウドのポータル画面からもISOを登録できます。

デスクトップ環境には[Cinammon](http://cinnamon.linuxmint.com/)用のISOも選べますが、[MATE](http://mate-desktop.org/)の方が軽くて好きなので、MATEのISOをダウンロードします。

* urlは、日本に近い台湾を選びました。
* ostypeidは、100の「Other Ubuntu (64-bit)」にしました。
* bootableは、trueにしてISOから起動できるようにします。

``` bash
$ idcf-compute-api registerIso \
 --displaytext=linuxmint-17-mate-dvd-64bit.iso \
 --name=linuxmint-17-mate-dvd-64bit.iso \
 --url=http://ftp.stust.edu.tw/pub/Linux/LinuxMint/isos//stable/17/linuxmint-17-mate-dvd-64bit.iso \
 --zoneid=1 \
 --ostypeid=100 \
 --bootable=true \
 --ispublic=false
```

### インスタンス作成

IDCFクラウドのポータル画面にログインして、登録したISOからインスタンスを作成します。
インスタンスが作成できたら、Webコンソールを起動してインストール作業を行います。

### LinuxMintのインストール

ライブDVDのLinuxMintが起動しているので、デスクトップ上の「Install Linux Mint」 のアイコンをダブルクリックします。
Webコンソールは動きが遅いので、あわてずゆっくり作業します。また、GUIのウィンドウは中央に開くので、Webコンソール用のブラウザ画面を大きくして操作します。

インスト－ルは特に難しくないのですが、以下のようにしています。

* 言語に日本語を選択
* キーボードレイアウトは日本語、日本語
* ユーザー作成は、「自動的にログインする」にチェック

`please remove installation media and close the tray (if any) then press ENTER:`のメッセージが表示されたら、ポータル画面からインスタンスを選択してISOをデタッチします。

Webコンソールに戻り、ENTERを入力すると再起動するので、しばらく待ってからOSインストールで作成したユーザーでログインします。

以降の作業はSSHから行いたいので、SSH環境をインストールします。xrdpから使うのでパスワード認証を有効にします。
``` bash
$ sudo apt-get install openssh-server
```

SSHクライアントから接続の確認ができたら、Webコンソールは閉じても大丈夫です。

### vimをインスト－ル

Ubuntuのデフォルトのnanoは苦手なので、vimをインストールします。
alternativesでは、`/usr/bin/vim.basic`を選択します。

``` bash
$ sudo apt-get install vim
$ sudo update-alternatives --config editor
```

### xrdpのセットアップ

最初にapt-getのミラー設定をして、updateします。
``` bash
$ sudo sed -i~ -e 's;http://archive.ubuntu.com/ubuntu;http://ftp.jaist.ac.jp/pub/Linux/ubuntu;' /etc/apt/sources.list.d/official-package-repositories.list
$ sudo apt-get update
```

xrdpをインストールします。
``` bash
$ sudo apt-get install xrdp vnc4server
```

デスクトップ環境用に、認証ファイルを作成します。
``` bash
$ touch ~/.Xauthority
```

ここで一度rebootしておきます。
``` bash
$ sudo reboot
```

### リモートデスクトップ接続

今回はWindowsから接続します。リモートデスクトップのオプションを開いて、画面の大きさを適当に調整します。
接続すると、認証後にデスクトップが表示されます。
LinuxMint17ではWindowsもOSXも、リモートデスクトップでホイールマウスも最初から使えるので便利です。

### OSXの場合Microsoft Remote Desktop Connection Client for Macを使う

OSXで使えるRDPクライアントは2種類ありますが、[Microsoft Remote Desktop Connection Client for Mac 2.1](http://www.microsoft.com/ja-jp/download/details.aspx?id=18140)をダウンロードしてインストールします。インストーラーを右クリックして開くを実行します。App Storeで配布されている[Microsoft Remote Desktop](https://itunes.apple.com/jp/app/microsoft-remote-desktop/id715768417)を使うと、日本語キーボードが正しく認識されず、\と_が入力できないので使い物になりません。


### 日本語環境の構築

ここからがいつも面倒なところです。
最初にlocaleを確認します。インストール後にはlocaleはja_JP.UTF-8になっています。

``` bash
$ locale
LANG=ja_JP.UTF-8
LANGUAGE=
LC_CTYPE="ja_JP.UTF-8"
LC_NUMERIC="ja_JP.UTF-8"
LC_TIME="ja_JP.UTF-8"
LC_COLLATE="ja_JP.UTF-8"
LC_MONETARY="ja_JP.UTF-8"
LC_MESSAGES="ja_JP.UTF-8"
LC_PAPER="ja_JP.UTF-8"
LC_NAME="ja_JP.UTF-8"
LC_ADDRESS="ja_JP.UTF-8"
LC_TELEPHONE="ja_JP.UTF-8"
LC_MEASUREMENT="ja_JP.UTF-8"
LC_IDENTIFICATION="ja_JP.UTF-8"
LC_ALL=
```

### キーボードから「\」と「_」を入力できるようにする

プログラマなのでバックスラッシュとアンダースコアが入力できないと、仕事になりません。

いろいろ試したのですが、キーボードモデルはデフォルトのpc105のまま使い、
[Ubuntu 12.04でxrdpで日本語キーボードを認識させる]( https://gist.github.com/devlights/6337527)を参考にしながら、xrdpのキーマップを変更します。
リモートデスクトップ接続の場合、X.orgの自動デバイス認識がどうしてもうまくできませんでした。

XKBMODELはpc105です。
``` bash /etc/default/keyboard
XKBMODEL="pc105"
XKBLAYOUT="jp"
XKBVARIANT=""
XKBOPTIONS=""
```

キーマップファイルをダウンロードして使えるようにします。
```
$ cd /etc/xrdp
$ sudo wget http://www.mail-archive.com/xrdp-devel@lists.sourceforge.net/msg00263/km-e0010411.ini
$ sudo mv km-e0010411.ini km-0411.ini 
$ sudo chmod 644 km-0411.ini
$ sudo ln -s km-0411.ini km-e0010411.ini
$ sudo ln -s km-0411.ini km-e0200411.ini
$ sudo ln -s km-0411.ini km-e0210411.ini
```

xrdpをrestartします。
``` bash
$ sudo service xrdp restart
```

### 入力メソッドにiBusを使う

LinuxMint16のときfcitxだとレイアウトが読み込めなかったので、今回もiBusと、エンジンにはGoogleのMozcを使います。
``` bash
$ sudo apt-get install ibus-mozc
$ sudo reboot
```

reboot後に、デスクトップ右下の入力モードで、「あ]が表示される、日本語-Mozcを選択します。
インプットメソッドの切り替えは、デフォルトで`Ctrl + Space`が割り当てられていますが、
`Dart Editor`を使う場合、このキーボードショートカットはコード補完と重なってしまうため、は削除しておきます。

OSXの場合「英数」キーが、Windowsの場合は「半角/全角」キーが切り替えのトグルになっているようです。

### ディレクトリ名を英語にする

日本語環境にすると、「デスクトップ」や「ダウンロード」などディレクトリ名も日本語になってしますので、英語表記に直します。

デスクトップにログインして、Terminalを開きコマンドを実行します。
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

### まとめ
特に問題もなく、IDCFクラウド上にLinuxMit17 MATEのリモートデスクトップ環境ができました。
以前に比べてかなり便利になった気がします。

これで`Dart Editor`を使う準備ができたので、次からはDartをGUI環境で開発する準備をしようと思います。
