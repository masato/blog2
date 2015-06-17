title: "Raspberry PiでEmacs 24.5とCaskを使う"
date: 2015-04-13 09:59:56
tags:
 - RaspberryPi
 - Emacs
 - Cask
 - init-loader
description: Raspberry Pi (Model B)は非力なので開発環境には向いていませんがデバイスを操作するPythonを書いているとEmacsが欲しくなります。apt-getでEmacs 24がインストールできないようなので、これも不向きですがRaspberry Pi上でEmacsをビルドして使うことにします。
---

Raspberry Pi (Model B)は非力なので開発環境には向いていませんがデバイスを操作するPythonを書いているとEmacsが欲しくなります。apt-getでEmacs 24がインストールできないようなので、これも不向きですがRaspberry Pi上でEmacsをビルドして使うことにします。


<!-- more -->

## Emacs 24.5.1


apt-getでインストールできるEmacsは23.4でした。

``` bash 
$ sudo apt-get update
$ apt-cache show emacs23
Package: emacs23
Version: 23.4+1-4
...
```

仕方がないのでEmacsの最新版をビルドすることにします。

### 必要なパッケージ

Emacsのビルドに必要なパッケージをインストールします。

``` bash
$ sudo apt-get update
$ sudo apt-get install build-essential automake texinfo libncurses5-dev
```

### インストール

24.5の最新のソースコードをダウンロードしてビルドします。ターミナル接続が主でGUIは不要のため`--without-x`フラグを付けてconfigureします。

``` bash
$ mkdir ~/src
$ cd !$
$ wget http://ftp.gnu.org/pub/gnu/emacs/emacs-24.5.tar.xz
$ tar -Jxvf emacs-24.5.tar.xz
$ cd emacs-24.5
$ ./configure --without-x
$ make
$ sudo make install
```

ビルドとインストールに1時間30分くらいかかります。気長に待ちます。

``` bash
$ which emacs
/usr/local/bin/emacs
$ emacs --version
GNU Emacs 24.5.1
```

## Cask

Emacsのパッケージ管理は[Cask](http://cask.github.io/)を使います。

### インストール

``` bash
$ curl -fsSL https://raw.githubusercontent.com/cask/cask/master/go | python
Cloning into '/home/pi/.cask'...
...
Successfully installed Cask!  Now, add the cask binary to your $PATH:
  export PATH="/home/pi/.cask/bin:$PATH"
```


``` bash
$ echo 'export PATH="/home/pi/.cask/bin:$PATH"' >> ~/.bashrc
$ source ~/.bashrc
```

``` bash
$ cask --version
0.7.3
```

### 設定

`~/.emacs.d`ディレクトリに移動してcaskの初期処理を行います。

``` bash
$ cd ~/.emacs.d/
$ cask init
```

自動生成されたCaskファイルを修正してとりあえず不要なパッケージは削除します。

```el ~/.emacs.d/Cask
(source gnu)
(source melpa)

(depends-on "cask")
(depends-on "init-loader")
(depends-on "multi-term")
```

[init-loader](https://github.com/emacs-jp/init-loader)を使うため`init.el`のrequireは少なくなります。

``` el  ~/.emacs.d/init.el
(require 'cask "~/.cask/cask.el")
(cask-initialize)

(require 'init-loader)
(setq init-loader-show-log-after-init nil)
(init-loader-load "~/.emacs.d/inits")
```

`cask install`コマンドを実行して、Caskファイルに定義したパッケージをインストールします。

``` bash
$ cask install
```



## init-loader

Emacsの設定ファイルの管理は[init-loader](https://github.com/emacs-jp/init-loader)を使います。設定ファイルを配置するディレクトリを作成します。

``` bash
$ mkdir -p ~/.emacs.d/inits
$ cd !$
```

最低限必要な基本的な設定ファイルを作成します。

``` bash
$ tree -L 2
.
├── Cask
├── init.el
└── inits
    ├── 00-keybindings.el
    ├── 01-menu.el
    ├── 02-files.el
    └── 03-multi-term.el
```

### 00-keybindings.el

削除キーとヘルプだけキーバインドを変更します。

``` el ~/.emacs.d/inits/00-keybindings.el
(define-key global-map "\C-h" 'delete-backward-char)
(define-key global-map "\M-?" 'help-for-help)
```

* C-hでbackspaceする
* M-?でヘルプを表示する

### 01-menu.el

GUIは使わないのでメニューを非表示にします。

``` el ~/.emacs.d/inits/01-menu.el
(menu-bar-mode 0)
```

### 02-files.el

ファイルの操作もバックアップファイルを作らないなど基本的な項目です。

```el ~/.emacs.d/inits/02-files.el
(setq backup-inhibited t)
(setq next-line-add-newlines nil)
(setq-default tab-width 4 indent-tabs-mode nil)
```

* バックアップファイルを作らない 
* バッファの最後でnewlineで新規行を追加するのを禁止する
* 4タブのスペース変換


### 03-multi-term.el

USB-TTLシリアル変換モジュールを経由してホストマシンからscreenで接続しているためウィンドウが1つしか開きません。[Multi Term](http://www.emacswiki.org/emacs/MultiTerm)を使い、Emacs上でコーディングをプログラムの実行をできるようにします。

``` el ~/.emacs.d/inits/03-multi-term.el
(dolist (dir (list
              "/sbin"
              "/usr/sbin"
              "/bin"
              "/usr/bin"
              "/usr/local/bin"
              (expand-file-name "~/bin")
              (expand-file-name "~/.emacs.d/bin")
              ))
 (when (and (file-exists-p dir) (not (member dir exec-path)))
   (setenv "PATH" (concat dir ":" (getenv "PATH")))
   (setq exec-path (append (list dir) exec-path))))

(require 'multi-term)
(setq multi-term-program shell-file-name)
```
