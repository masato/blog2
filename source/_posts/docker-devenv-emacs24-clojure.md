title: "Docker開発環境をつくる - Emacs24とClojure"
date: 2014-09-20 00:40:32
tags:
 - DockerDevEnv
 - Clojure
 - Leiningen
 - Emacs
 - REPL
description: 初めは括弧ばかりで取り付きにくいClojureですが、慣れてくると括弧が気にならなくなるので不思議です。Emacsの開発環境を用意してClojureの勉強を進めたいと思います。使い捨てのDocker開発環境でもEmacsの設定まで用意しておくと気軽にコードを書けます。REPL専用のコンテナを作ってもいいかも。
---

* `Update 2014-10-04`: [Docker開発環境をつくる - Emacs24とEclimからEclipseに接続するJava開発環境](/2014/10/04/docker-devenv-emacs24-eclim-java/)
* `Update 2014-09-29`: [Dockerデータ分析環境 - Part8: Gorilla REPLとCMDのcdでno such file or directoryになる場合](/2014/09/29/docker-analytic-sandbox-gorilla-repl/)

初めは括弧ばかりで取り付きにくいClojureですが、慣れてくると括弧が気にならなくなるので不思議です。
Emacsの開発環境を用意してClojureの勉強を進めたいと思います。
使い捨てのDocker開発環境でもEmacsの設定まで用意としておくと気軽にコードを書けます。REPL専用のコンテナを作ってもいいかも。

<!-- more -->

### プロジェクトの作成

プロジェクトを作成します。

``` bash
$ mkdir ~/docker_apps/clojure
$ cd !$
```

最初にプロジェクト全体のファイル構成です。

``` bash
~/docker_apps/clojure
├── .emacs.d
│   ├── Cask
│   ├── init.el
│   └── inits
│       ├── 00-keybinds.el
│       ├── 01-files.el
│       └── 02-clojure-mode.el
├── .lein
│   └── profiles.clj
└── Dockerfile
```

### Cask

[Emacsの構成](/2014/09/19/docker-devenv-emacs24-init-loader-cask-pallet/)はほとんど変わりません。

clojure用パッケージはたくさんありますが、基本の3つをインストールします。

* [clojure-mode](https://github.com/clojure-emacs/clojure-mode)
* [CIDER](https://github.com/clojure-emacs/cider)
* [smartparens](https://github.com/Fuco1/smartparens)

``` el ~/docker_apps/clojure/Cask
(source gnu)
(source marmalade)
(source melpa)

(depends-on "pallet")
(depends-on "init-loader")

;; clojure
(depends-on "clojure-mode")
(depends-on "cider")
(depends-on "smartparens")
```

### init.elとinitsファイル

init.elでcaskとpalletをロードします。

``` el ~/docker_apps/clojure/init.el
(require 'cask "~/.cask/cask.el")
(cask-initialize)
(require 'pallet)

(require 'init-loader)
(setq init-loader-show-log-after-init nil)
(init-loader-load "~/.emacs.d/inits")
```

init-loader.elで分割したキーバインドです。

``` el ~/docker_apps/clojure/inits/00-keybinds.el
(define-key global-map "\C-h" 'delete-backward-char)
(define-key global-map "\M-?" 'help-for-help)
```

同じくファイル系の設定です。

``` el ~/docker_apps/clojure/inits/01-files.el
(setq backup-inhibited t)
(setq next-line-add-newlines nil)
(setq-default tab-width 4 indent-tabs-mode nil)
```

Clojureの設定は、smartparensを使います。

``` el ~/docker_apps/clojure/inits/02-clojure-mode.el
(add-hook 'clojure-mode-hook 'smartparens-strict-mode)
```

### $HOME/.lein/profiles.clj

[Leiningen](https://github.com/technomancy/leiningen)の設定ファイルです。Leiningen経由でCIDERを使う場合[cider-nrepl](https://github.com/clojure-emacs/cider-nrepl)のREADME.mdには以下のように書いてありますが、このままだとエラーになります。(2014-09-20現在)

``` el
{:user {:plugins [[cider/cider-nrepl "0.7.0"]]}}
```

`M-x cider-jack-in`を実行すると以下のエラーが表示されます。

``` bash
; CIDER 0.8.0alpha (package: 20140916.627) (Java 1.7.0_65, Clojure 1.6.0, nREPL 0.2.6)
WARNING: The following required nREPL ops are not supported:
ns-list ns-vars undef
Please, install (or update) cider-nrepl 0.8.0-snapshot and restart CIDER
WARNING: CIDER's version (0.8.0-snapshot) does not match cider-nrepl's version (0.7.0)
user>
```

今のところ以下のように、`0.8.0-SNAPSHOT`を指定します。SBTでも似たようなことがあった気がします。Mavenって。

``` clj ~/docker_apps/clojure/.lein/profiles.clj
{:user {:plugins [[cider/cider-nrepl "0.8.0-SNAPSHOT"]]}}
```

### Dockerfile

Caskのインストールまでは、[EmacsのDokcerfile](/2014/09/19/docker-devenv-emacs24-init-loader-cask-pallet/)と同じです。
ClojureのインストールはJDKとLeiningenをインストールするだけなので簡単です。

``` bash ~/docker_apps/clojure/Dockerfile
FROM ubuntu:14.04
MAINTAINER Masato Shimizu <ma6ato@gmail.com>

ENV HOME /root

## apt-get update
RUN sed -i~ -e 's/archive.ubuntu.com/ftp.jaist.ac.jp/' /etc/apt/sources.list \
 && apt-get update

## Japanese Environment
RUN DEBIAN_FRONTEND=noninteractive apt-get install -y language-pack-ja
ENV LANG ja_JP.UTF-8
RUN update-locale LANG=ja_JP.UTF-8
RUN mv /etc/localtime /etc/localtime.org
RUN ln -s /usr/share/zoneinfo/Asia/Tokyo /etc/localtime

RUN DEBIAN_FRONTEND=noninteractive \
    apt-get install -y build-essential software-properties-common \
                       zlib1g-dev libssl-dev libreadline-dev libyaml-dev \
                       libxml2-dev libxslt-dev sqlite3 libsqlite3-dev \
                       emacs24-nox emacs24-el git byobu wget curl unzip tree\
                       python
WORKDIR /root

# Cask
RUN curl -fsSkL https://raw.github.com/cask/cask/master/go | python
ADD .emacs.d /root/.emacs.d/
RUN echo 'export PATH="/root/.cask/bin:$PATH"' >> /root/.profile && \
    /bin/bash -c 'source /root/.profile && cd /root/.emacs.d && cask install'
ENV PATH /root/.cask/bin:$PATH

# Clojure
ADD .lein/profiles.clj /root/.lein/
RUN DEBIAN_FRONTEND=noninteractive \
  apt-get install -y openjdk-7-jdk
RUN curl -s https://raw.githubusercontent.com/technomancy/leiningen/2.5.0/bin/lein > \
  /usr/local/bin/lein && \
  chmod 0755 /usr/local/bin/lein
ENV LEIN_ROOT 1
RUN lein

# Define default command.
CMD ["bash"]

# Clean
RUN apt-get clean && rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*
```

### build && run

``` bash
$ cd ~/docker_apps/clojure
$ docker build -t masato/clojure .
$ docker run -it --rm masato/clojure
```

### Leiningenのプロジェクトとclojure-mode

`lein new`コマンドでプロジェクトを作成します。

``` bash
$ lein new hello
```

作成されたcljファイルをEmacsで開きます。

``` bash
$ emacs hello/src/hello/core.clj
```

### REPLの起動

Emacsで開いたleinプロジェクトからnREPLサーバーへ接続する場合は`M-x cider-jack-in`または`C-c M-j`を実行します。
普通に`lein repl`で起動するよりもREPLが使いやすいです。

``` bash
; CIDER 0.8.0alpha (package: 20140919.457) (Java 1.7.0_65, Clojure 1.6.0, nREPL 0.2.6)
user>
```

### REPLの終了

以下のどちらかを実行してREPLを終了します。

```
C-c C-q 
```

または

```
M-x cider-quit
```



