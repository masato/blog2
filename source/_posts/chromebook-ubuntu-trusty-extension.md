title: "HP Chromebook 11にUbuntu 14.04.1とXfceをインストールする"
date: 2015-02-05 20:59:58
tags:
 - Chromebook
 - Ubuntu
 - crouton
 - Xfce
description: 去年購入したHP Chromebook 11はしばらく放置していたのですが、BeagleBone BlackとUSB-EthernetやUSB-Serial接続して開発用の端末にしよう考えています。ChromeOSのネイティブウインドウ上でX11を操作できるcrouton integrationなど、/2014/05/02/chromebook-ubuntuとはすこし違った方法でUbuntuを再インストールしようと思います。
---

去年購入したHP Chromebook 11はしばらく放置していたのですが、BeagleBone BlackとUSB-EthernetやUSB-Serial接続して開発用の端末にしよう考えています。ChromeOSのネイティブウインドウ上でX11を操作できる[crouton integration](https://chrome.google.com/webstore/detail/crouton-integration/gcpneefbbnfalgjniomfjknbcgkbijom)など、[以前](/2014/05/02/chromebook-ubuntu/)とはすこし違った方法でUbuntuを再インストールしようと思います。

<!-- more -->

## VefificationはOFF、Developper Mode

`ESC + Refresh + Powerボタン`を押してChromebookをリカバリモードで起動します。VefificationがOFFの状態であることを確認して、ctrl+dでDevelopper Modeに入ります。言語とキーボードは以下のようにします。

* 言語の選択: 日本語
* キーボードの選択: 英語(米国)のキーボード


## croutonからUbuntuのインストール

ChromeOSが起動したらアップデートを確認して適用します。

* 設定 > ChromeOSについて
* バージョン: 40.0.2214.93

Chromeブラウザから`ctrl + alt + t`をタイプしてcroshを起動します。

``` bash
crosh> shell
chronos@localhost / $ 
```

すでに[crouton](https://github.com/dnschneid/crouton)でインストールしたchrootをすべて削除する場合は次を実行します。

``` bash
$ cd /usr/local/chroots
$ sudo delete-chroot * 
$ sudo rm -rf /usr/local/bin
```

[crouton](https://github.com/dnschneid/crouton)をダウンロードします。

``` bash
$ wget http://goo.gl/fd3zc -O ~/Downloads/crouton
```

`-r`フラグでリリースはUbuntu 14.04 trustyを選択します。

``` bash
$ sudo sh -e ~/Downloads/crouton -r trusty -t xfce,keyboard,audio,cli-extra,extension,xiwi,chromium
```

インストールには30分くらいかかります。

### Chromiumターゲット

targetフラグにchromeを指定してもARMにChromeはまだビルドされていなので、Chromiumを指定します。

> Google Chrome does not yet have an ARM build. Installing Chromium instead.


### crouton integrationターゲット

[crouton integration](https://chrome.google.com/webstore/detail/crouton-integration/gcpneefbbnfalgjniomfjknbcgkbijom)をChromeウェブストアからインストールしておきます。croutonの`-t`フラグのターゲットにxiwiフラグを追加するとChromeブラウザの拡張機能としてcroutonを別ウィンドウとして使えるようになります。


### CLIの起動

croshからUbuntuをマウントしてCLI接続します。

``` bash
$ sudo enter-chroot
Entering /mnt/stateful_partition/crouton/chroots/trusty...
(trusty)mshimizu@localhost:~$
```

croshはまだ日本語の入力ができないのですが、コンソールからUbuntuを使う場合はChromeブラウザから操作できます。

### Xfceの起動

croshからUbuntuとXfceのデスクトップをマウントします。

``` bash
$ sudo startxfce4
```

ChromeOSとUbuntuを切り替える場合は、以下のキーを押します。

* ChromeSへ切り換え: Ctrl+Shift+Alt+Back(F1)
* Ubuntuへ切り換え: Ctrl+Shift+Alt+Forward(F2)

croutonにxiwiとextensionターゲット、Chromeブラウザに[crouton integration](https://chrome.google.com/webstore/detail/crouton-integration/gcpneefbbnfalgjniomfjknbcgkbijom)をインストールしているので全画面表示とウィンドウ表示を切り替えることができます。

![crouton-integration.png](/2015/02/05/chromebook-ubuntu-trusty-extension/crouton-integration.png)

Xfceのバージョンを確認します。ウインドウが表示されます。バージョンは4.10です。

``` bash
$ xfce4-abount
```

## Ubuntuの日本語設定

日本語でUbuntuとXfceを使うための設定をしていきます。

### フォント

Takaoフォントをインストールします。

``` bash
$ sudo apt-get install fonts-takao
```

### locale

localeを更新します。

``` bash
$ export LANG=ja_JP.UTF-8
$ sudo locale-gen $LANG
$ sudo update-locale $LANG
```

外観の設定でフォント指定を変更します。

* Settings > Appereance > Fonts > TakaoPゴシック > 12 > Enable anti-aliasing

### 日本語入力

Mozcの日本語入力をインストールします。

``` bash
$ sudo apt-get install ibus ibus-mozc
```

* Settings > Keyboard Input Methods > Input Methods 

`Customize active input methods`にチェックをいれて、AddボタンからMozcを追加します。

* Add > Japanese > Mozc

デフォルトで英語と日本語の切り換えは`ctrl + space`でトグルできます。Emacsの`mark set`と競合してしまうのでKeyboard Shortcutsの設定を変更します。

* Settings > Keybord Input Methods > Keyboard Shortcuts > <Control><Alt>space

## エディタ

デフォルトのnanoから、vimにエディタを変更します。

``` bash
$ sudo apt-get install vim
$ sudo update-alternatives --set editor /usr/bin/vim.basic
```

