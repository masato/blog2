title: 'ChromebookにUbuntuをインストール'
date: 2014-05-02 02:08:45
tags:
 - Chromebook
 - Ubuntu
 - Linux-ARM
description: HP Chromebook 11をオフラインでも使えるように、croutonを使いUbuntu Trustyをインストールします。デスクトップ環境には軽量なXfceをインスト－ルして、日本語環境を構築します。
---

* Update: [HP Chromebook 11にUbuntu 14.04.1とXfceをインストールする](/2015/02/05/chromebook-ubuntu-trusty-extension/)

`HP Chromebook 11`をオフラインでも使えるように、[crouton](https://github.com/dnschneid/crouton)を使い`Ubuntu Trusty`をインストールします。デスクトップ環境には軽量なXfceをインスト－ルして、日本語環境を構築します。

<!-- more -->

### Chromebookをデベロッパーモードで起動する

Chromebookを`「ESC」+「Refresh」+「Power button」`を押してリカバリモードで起動します。
リカバリーモードにはいり、「Ctrl」+「D」+「Enter」を押し、そのままの状態でしばらく待つと、ビープ音が鳴り、デベロッパーモードに入る準備が始まります。
再起動が繰り返されるので、10分くらい待ちます。

初期設定とGoogleアカウントを入力します。
* 言語の選択: 日本語
* キーボードの選択: 英語(米国)のキーボード

### Ubuntuのインストール

「Ctrl」+「Alt」+「T」でターミナル起動して、shellを実行します。

``` bash
crosh > shell
```
croutonをダウンロードして`Ubuntu trusty (14.04)` をインストール。

``` bash
$ wget http://goo.gl/fd3zc -O ~/Downloads/crouton
$ sudo sh -e ~/Downloads/crouton -r trusty -t xfce,cli-extra,keyboard
$ sudo startxfce4
```
Xfceが起動したら、ターミナルを開きバージョンを確認します。

``` bash
$ cat /etc/lsb-release
DISTRIB_ID=ubuntu
DISTRIB_RELEASE=14.04
DISTRIB_CODENAME=trusty
DISTRIB_DESCRIPTION="Ubuntu 14.04 LTS"
```

### ChromeOSとUbuntuを切り替える

以下のキーでChromeOSとUbuntuを切り替えることができます。
「Ctrl」+「Shift」+「Alt」+「Back(F1)」
「Ctrl」+「Shift」+「Alt」+「Forward(F2)」

### ユーザー管理

adminグループを作成して、作業ユーザーを管理者グループにします。

``` bash
$ sudo groupadd admin
$ sudo usermod -aG admin masato
$ sudo -i
# visudo
%sudo   ALL=(ALL:ALL) ALL
%admin ALL=(ALL) NOPASSWD:ALL
```

### Chromiumブラウザ

Linux-ARMにはChromeパッケージがないため、Chromiumブラウザをインストールします。

``` bash
$ sudo apt-get update
$ sudo apt-get install chromium-browser
$ sudo update-alternatives --config x-www-browser
```

### Ubuntuの日本語環境

日本語環境の構築が終わったら、`「Ctrl」+「Shift」+「Alt」+「Back(F1)」`で、ChromeOSに戻ります。
`「Ctrl」+「C」`でcroutonを停止し、Xfceを再起動します。

``` bash
$ sudo apt-get update
$ wget -q https://www.ubuntulinux.jp/ubuntu-ja-archive-keyring.gpg -O- | sudo apt-key add -
$ wget -q https://www.ubuntulinux.jp/ubuntu-jp-ppa-keyring.gpg -O- | sudo apt-key add -
$ sudo wget https://www.ubuntulinux.jp/sources.list.d/trusty.list -O /etc/apt/sources.list.d/ubuntu-ja.list
$ sudo apt-get update
$ sudo apt-get upgrade
$ sudo apt-get install language-pack-ja
$ export LANG=ja_JP.UTF-8
$ sudo update-locale LANG=ja_JP.UTF-8
$ sudo apt-get install language-selector-gnome
```

Takaoフォトをインストール。

``` bash
$ sudo apt-get install fonts-takao
```
フォントの設定。

```
設定 > 外観 > フォント > TakaoPゴシック > 12 > アンチエイリアス
```

ibusとmozcをインストール。

``` bash
$ sudo apt-get update
$ sudo apt-get install zenity
$ sudo apt-get install libglib2.0-bin
$ sudo apt-get install ibus ibus-mozc
```

### vim

vim.basic インストール。

``` bash
$ sudo apt-get install vim
$ sudo update-alternatives --config editor
alternative editor (/usr/bin/editor を提供) には 2 個の選択肢があります。

  選択肢    パス              優先度  状態
------------------------------------------------------------
* 0            /usr/bin/vim.basic   30        自動モード
  1            /usr/bin/vim.basic   30        手動モード
  2            /usr/bin/vim.tiny    10        手動モード

現在の選択 [*] を保持するには Enter、さもなければ選択肢の番号のキーを押してください: 
```

.vimrcの設定をします。

``` bash
$ mkdir ~/tmp
```

`~/.vimrc`

``` 
set backupdir=~/tmp
```

### まとめ

Chromebookはオンラインで使うことが前提のため、オフラインになるとほとんど使い物になりません。
`BeagleBone Black`などと同様のLinux-ARMのハードウェアとして、UbuntuやDebianがインストールできます。
ChromeOSはモダンで美しいOSなので、Xfceは物足りなく感じますが、ローカルでEmacsが使えるのはやはり便利です。
