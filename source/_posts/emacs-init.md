title: 'Ubuntu 14.04にEmacs24をインストールする'
date: 2014-04-27 19:09:14
tags:
 - Ubuntu
 - Emacs
description: 昔は新しい開発環境を作ると.emacsを書くのが面倒でしたが、Emacs24で標準になったpackage.elを使うとパッケージのインストールも簡単で、すぐにプログラムを始めることができます。Ubuntu 14.04にEmacs 24.3.1をインストールします。
---

* `Update 2014-09-19`: [Docker開発環境のEmacs24をinit-loader.elとCaskとPalletでパッケージ管理する](/2014/09/19/docker-devenv-emacs24-init-loader-cask-pallet/)


昔は新しい開発環境を作ると.emacsを書くのが面倒でしたが、Emacs24で標準になったpackage.elを使うと
パッケージのインストールも簡単で、すぐにプログラムを始めることができます。
`Ubuntu 14.04`に`Emacs 24.3.1`をインストールします。

<!-- more -->

### Emacs24のインストール

Ubuntuのバージョンを確認します。

``` bash
# cat /etc/lsb-release
DISTRIB_ID=Ubuntu
DISTRIB_RELEASE=14.04
DISTRIB_CODENAME=trusty
DISTRIB_DESCRIPTION="Ubuntu 14.04 LTS"
```

Emacsのインストール

``` bash
# apt-get update
# apt-get install emacs24-nox emacs24-el 
# emacs -version
GNU Emacs 24.3.1
```

### init-loader.el

[init-loader](http://melpa.milkbox.net/#/init-loader)は[MELPA](http://melpa.milkbox.net/)からダウンロードするので、package-user-dirにリポジトリを追加します。
[Marmalade](http://marmalade-repo.org/)も一緒に追加しておきます。

``` el ~/.emacs.d/init.el
;; package
(require 'package)
(add-to-list 'package-archives '("melpa" . "http://melpa.milkbox.net/packages/") t)
(add-to-list 'package-archives '("marmalade" . "http://marmalade-repo.org/packages/"))
(package-initialize)
```

`~/.emacs.d/init.el`の設定を分割して管理できるように、[init-loader.el](https://github.com/emacs-jp/init-loader)をインストールします。

`M-x list-packages`を表示し、init-loaderを検索、選択(i)して、実行(x)します。

分割した設定ファイルを配置するディレクトリを作成します。

``` bash
$ mkdir ~/.emacs.d/inits
```

init.elを作成します。

``` el ~/.emacs.d/init.el
(require 'init-loader)
(setq init-loader-show-log-after-init nil)
(init-loader-load "~/.emacs.d/inits")
```

`M-x eval-buffer`でバッファを実行します。

### multi-term

Emacs上でシェルを実行するため、[multi-term](http://www.emacswiki.org/emacs/MultiTerm)をインストールします。
`M-x list-packages`を表示し、multi-termを検索、選択(i)して、実行(x)します。

先ほど作成した`~/.emacs.d/inits`ディレクトリの下に、個別の設定ファイルを書いていきます。

``` el  ~/.emacs.d/inits/03-multi-term.el
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

;;multi-term
(setq load-path(cons "~/.emacs.d/" load-path))
(setq multi-term-program "/bin/bash")
(require 'multi-term)
```

`M-x eval-buffer`でバッファを実行します。

### multi-termの使い方

`M-x multi-term`で、シェルが起動します。
Tab補完も効くので、とても便利です。
