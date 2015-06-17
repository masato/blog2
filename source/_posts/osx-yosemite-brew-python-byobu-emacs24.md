title: "OSXにHomebrewでtmuxとEmacs 24.4をインストールする"
date: 2015-03-19 22:42:50
tags:
 - Emacs
 - Cask
 - OSX
 - Homebrew
 - byobu
 - tmux
description: 普段はクラウド上のUbuntuコンテナやLinux Mintをメインの作業環境にしています。最近Arduinoなどマイコンを操作していると、USBシリアル通信をするホストマシンとしてOSXで一通りの開発環境が必要になりました。いつものクラウド上と同じようなの開発環境とOSX上にも用意しようと思います。
---

普段はクラウド上のUbuntuコンテナやLinux Mintをメインの作業環境にしています。最近Arduinoなどマイコンを操作していると、USBシリアル通信をするホストマシンとしてOSXで一通りの開発環境が必要になりました。いつものクラウド上と同じようなの開発環境とOSX上にも用意しようと思います。

<!-- more -->

## ターミナルマルチプレクサ

### byobu-configが起動しない

UbuntuやLinix Mintではターミナルマルチプレクサに[byobu](http://byobu.co/)を使っています。OSXにもHomebrewでインストールしたかったのですが、byobu-configが起動しません。

``` bash
$ brew update
$ brew install byobu
$ byobu-config
ERROR: Could not import the python snack module
```

原因は`newt`パッケージとPythonのモジュールにありそうです。

```
$ brew info newt
newt: stable 0.52.18 (bottled)
https://fedorahosted.org/newt/
Not installed
From: https://github.com/Homebrew/homebrew/blob/master/Library/Formula/newt.rb
==> Dependencies
Required: gettext ✔, popt ✔, s-lang ✔
==> Options
--with-python
        Build with python support
```

[byobu: Python crashes when executing the program on 10.9 #30252](https://github.com/Homebrew/homebrew/issues/30252)のissueを読むと、`--with-python`オプションを付けてビルドすればよいとか、単にupgradeするだけで良いとか嫌な予感がします。

``` bash
$ brew update && brew upgrade
$ brew remove newt && brew install newt --with-python
```

newtの最新0.52.18のFormulaを読むと指定したオプションは使われないようです。そもそもPythonがクラッシュするissueの対応なので、OSX版のbyobuは現在不安定な感じです。

``` ruby /usr/local/Library/Formula/newt.rb
class Newt < Formula
  homepage 'https://fedorahosted.org/newt/'
  url 'https://fedorahosted.org/releases/n/e/newt/newt-0.52.18.tar.gz'
...
  def install
    args = ["--prefix=#{prefix}", "--without-tcl"]
    args << "--without-python" if build.without? 'python'
...
      # don't link to libpython.dylib
      # causes https://github.com/Homebrew/homebrew/issues/30252
      # https://bugzilla.redhat.com/show_bug.cgi?id=1192286
      s.gsub! "`$$pyconfig --ldflags`", '"-undefined dynamic_lookup"'
      s.gsub! "`$$pyconfig --libs`", '""'
....
```

### tmux

byobuの場合でもいつも[tmux](http://tmux.sourceforge.net/)だけなので直接使うことにします。byobuと一緒にtmuxが入っていますが再インストールしました。

``` bash
$ brew remove byobu
$ brew remove tmux && brew install tmux
$ tmux -V
tmux 1.9a
```

最近はあまりカスタマイズしないで使うようにしているので`.tmux.conf`もシンプルです。

* prifixのキーバインドを`C-t`に変更
* コピーモードは`C-t s`で入る

```bash ~/.tmux.conf
set -g prefix C-t
unbind C-b
bind C-t send-prefix
bind s copy-mode
```

ウインドウの移動ができることを確認します。

* 前のウィンドウ移動: C-t p
* 次のウィンドウ移動: C-t n

`C-t s`でコピーモードに入りスクロール移動ができることを確認します。

* 上にスクロール移動: Up Arrow
* 下にスクロール移動: Down Arrow
* 1画面上にスクロール移動: Shift + fn + Up Arrow
* 1画面下にスクロール移動: Shift + fn + Down Arrow
* コピーモードを出る: q

## Emacs

OSX YosemiteにデフォルトでインストールされているEmacsは22.1.1と少し古いです。

``` bash
$ emacs --version
GNU Emacs 22.1.1
```

HomebrewからEmacsをインストールします。`--with-cocoa`オプションを付けてGUIのEmacs.appも合わせてビルドします。

``` bash
$ brew update
$ brew install --with-cocoa emacs
```

バイナリのインストール先とバージョンを確認します。

``` bash
$ ls -l /usr/local/bin/emacs
lrwxr-xr-x  1 mshimizu  admin  30  3 19 16:43 /usr/local/bin/emacs -> ../Cellar/emacs/24.4/bin/emacs
$ which emacs
/usr/local/bin/emacs
$ emacs --version
GNU Emacs 24.4.1
```

### Emacs.app

Emacs.appを/Applicationsにシムリンクを作ります。

```
$ brew linkapps emacs
Linking /usr/local/opt/emacs/Emacs.app
Finished linking. Find the links under /Applications.
```

GUIのEmacs.appの起動を確認します。

``` bash
$ open /Applications/Emacs.app
```

## Cask

Emacsのパッケージ管理は[Cask](https://github.com/cask/cask)を使います。OSXなのでHomebrewでもインストールできます。

``` bash
$ brew install cask
$ cask --version
0.7.2
```

### Caskファイル

`~/.emacs.d`ディレクトリに移動してCaskファイルを作成します。

``` bash
$ cd ~/.emacs.d/
$ cask init
```


作成されたCaskファイルにはパッケージがたくさん書いてあります。最初に必要なパッケージだけインストールします。[Pallet](https://github.com/rdallasgray/pallet)、[init-loader](https://github.com/emacs-jp/init-loader)、[js2-mode](https://github.com/mooz/js2-mode)だけ定義します。

```el ~/.emacs.d/Cask
(source gnu)
(source melpa)

(depends-on "pallet")
(depends-on "init-loader")

;; js2-mode
(depends-on "js2-mode")
(depends-on "ac-js2")
```

### elispファイル

`init.el`の最初でcask.elを読み込みます。Homebrewでインストールした場合は、`/usr/local/Cellar/cask/{version}/cask.el`にあります。

```el ~/.emacs.d/init.el
(require 'cask "/usr/local/Cellar/cask/0.7.2/cask.el")
(cask-initialize)
(require 'pallet)
(require 'init-loader)
(pallet-mode t)

(setq init-loader-show-log-after-init nil)
(init-loader-load "~/.emacs.d/inits")
```

init-loaderを使いelispは`~/.emacs.d/inits`に分割して書きます。

```bash
$ mkdir -p ~/.emacs.d/inits
```

削除キーとヘルプだけキーバインドを変更します。

```el ~/.emacs.d/inits/00-keybindings.el
(define-key global-map "\C-h" 'delete-backward-char)
(define-key global-map "\M-?" 'help-for-help)
```

CUIがメインでメニューは邪魔なので消しました。

```el ~/.emacs.d/inits/01-menu.el
(menu-bar-mode 0)
```

ファイルはバックアップを作らなかったり、タブの指定などを少し書きます。

```el ~/.emacs.d/inits/02-files.el
(setq backup-inhibited t)
(setq next-line-add-newlines nil)
(setq-default tab-width 4 indent-tabs-mode nil)
```

Node.jsのプログラミング用に[js2-mode](https://github.com/mooz/js2-mode)の設定をします。

```el ~/.emacs.d/inits/03-js2-mode.el
(autoload 'js2-mode "js2-mode" nil t)
(add-to-list 'auto-mode-alist '("\\.js$" . js2-mode))
(add-hook 'js2-mode-hook
          '(lambda ()
             (setq js2-basic-offset 4)))

(add-hook 'js-mode-hook 'js2-minor-mode)
(add-hook 'js2-mode-hook 'ac-js2-mode)
```

### cask install

最終的に以下のようなファイルを作成しました。

``` bash
$ tree -L 2
.
├── Cask
├── init.el
└── inits
    ├── 00-keybindings.el
    ├── 01-menu.el
    ├── 02-files.el
    └── 03-js2-mode.el
```

`cask install`コマンドを実行して、Caskに定義したパッケージをインストールします。

``` bash
$ cask install
```



