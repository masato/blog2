title: "CIDERのcompleteionにac-nreplはdeprecatedなのでac-ciderを使う"
date: 2014-09-24 00:05:23
tags:
 - DockerDevEnv
 - Emacs
 - Clojure
 - REPL
description: 以前Eclipseが開発環境であったときは自動補完ばかり使っていました。EmacsのCUI開発環境に切り替えてからは自動補完にあまり魅力を感じていないのですが、あれば便利な機能です。EmacsとClojureの開発環境にcompletionを導入しようとして調べていると、ac-nreplはdeprecatedなので、最近はac-ciderを使うようです。Clojureが人気になっているのか、もともとなのかわかりませんが、EmacsのClojureまわりのパッケージは動きが多くて、ついていくのが大変です。
---

以前Eclipseが開発環境であったときは自動補完ばかり使っていました。EmacsのCUI開発環境に切り替えてからは自動補完にあまり魅力を感じていないのですが、あれば便利な機能です。

[EmacsとClojureの開発環境](/2014/09/20/docker-devenv-emacs24-clojure/)にcompletionを導入しようとして調べていると、[ac-nrepl](https://github.com/clojure-emacs/ac-nrepl)はdeprecatedなので、最近は[ac-cider](https://github.com/clojure-emacs/ac-cider)を使うようです。

Clojureが人気になっているのか、もともとなのかわかりませんが、EmacsのClojureまわりのパッケージは動きが多くて、ついていくのが大変です。

<!-- more -->

### Emacsの設定追加

Caskファイルに、`auto-complete`と`ac-cider`を追加します。

``` el ~/docker_apps/clojure/.emacs.d/Cask
(source gnu)
(source marmalade)
(source melpa)

(depends-on "pallet")
(depends-on "init-loader")

;; clojure
(depends-on "clojure-mode")
(depends-on "cider")
(depends-on "smartparens")
(depends-on "auto-complete")
(depends-on "ac-cider")
```

initsに[ac-ciderのREADME.md](https://github.com/clojure-emacs/ac-cider)にあるように、`ac-cider`の起動に必要な設定を書きます。

TABキーで`auto-complete`を実行する設定も追加しました。Eclipeみたいに候補がインラインでリストされると思ったのですが、インラインで候補が自動的に補完されるようです。

``` el  ~/docker_apps/clojure/.emacs.d/inits/03-ac-cider.el

(require 'auto-complete-config)
(ac-config-default)

(require 'ac-cider)
(add-hook 'cider-mode-hook 'ac-flyspell-workaround)
(add-hook 'cider-mode-hook 'ac-cider-setup)
(add-hook 'cider-repl-mode-hook 'ac-cider-setup)

(eval-after-load "auto-complete"
  '(add-to-list 'ac-modes 'cider-mode))
(defun set-auto-complete-as-completion-at-point-function ()
  (setq completion-at-point-functions '(auto-complete)))

(add-hook 'auto-complete-mode-hook 'set-auto-complete-as-completion-at-point-function)
(add-hook 'cider-mode-hook 'set-auto-complete-as-completion-at-point-function)
```

### 起動確認

`ac-nrepl`がインストールされると、`cider-mode`のバッファのタイトルにACが付きます。
今回は[smartparens](https://github.com/Fuco1/smartparens)も使っているので、表示は`(Clojure SP/s AC)`となりました。

### The Joy of Clojure, Second Editionで勉強する

読みたかった[The Joy of Clojure, Second Edition](http://www.manning.com/fogus2/)ですがManningのDOTDが来たのでタイミング良く買えました。最新の`Clojure 1.6`をカバーしているのでテキストとして最適です。

DockerコンテナでEmacsを使ったClojureの開発環境もだいたい出来上がってきました。
メインの開発言語にするくらいの気持ちで勉強していきます。

