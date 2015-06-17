title: "Docker開発環境のEmacs24をinit-loader.elとCaskとPalletでパッケージ管理する"
date: 2014-09-19 00:02:35
tags:
 - DockerDevEnv
 - Ubuntu
 - Emacs
 - Cask
 - Pallet
description: いままではinit-loader.elとpackage.elを使ってEmacs24のパッケージ管理を行っていました。最近はCaskとPalletを使うのが良いらしいので追加して使ってみます。package.elもそのまま使えてパッケージ管理がさらに楽になります。
---

* `Update 2014-10-04`: [Docker開発環境をつくる - Emacs24とEclimからEclipseに接続するJava開発環境](/2014/10/04/docker-devenv-emacs24-eclim-java/)
* `Update 2014-09-24`: [CIDERのcompleteionにac-nreplはdeprecatedなのでac-ciderを使う](/2014/09/24/ac-nrepl-deprecated-using-ac-cider/)
* `Update 2014-09-20`: [Docker開発環境をつくる - Emacs24とClojure](/2014/09/20/docker-devenv-emacs24-clojure/)

いままでは[init-loader.elとpackage.elを使って](/2014/04/27/emacs-init/)Emacs24のパッケージ管理を行っていました。
最近は[Cask](https://github.com/cask/cask)と[Pallet](https://github.com/rdallasgray/pallet)を使うのが良いらしいので追加して使ってみます。package.elもそのまま使えてパッケージ管理がさらに楽になります。

<!-- more -->


### プロジェクトの作成

プロジェクトを作成します。

``` bash
$ mkdir ~/docker_apps/emacs
$ cd !$
```

最初にプロジェクト全体のファイル構成です。

``` bash
~/docker_apps/emacs
├── .emacs.d
│   ├── Cask
│   ├── init.el
│   └── inits
│       ├── 00-keybinds.el
│       └── 01-files.el
└── Dockerfile
```

### CaskとPallet

[Pallet](https://github.com/rdallasgray/pallet)のREADME.mdに従ってインストールします。
Palletの先にCaskをインストールしますが、`cask init`でCaskファイルは作成せずに、palletだけ定義したCaskファイルを用意して使います。

Caskファイルは、`~/.emacs.d/Cask`に配置します。
`cask init`を実行するときは、`~/.emacs.d/`ディレクトリに移動してからコマンドを実行します。
init-loaderもインストールして、init.elを分割管理できるようにします。

``` bash ~/docker_apps/emacs/.emacs.d/Cask
(source gnu)
(source marmalade)
(source melpa)

(depends-on "pallet")
(depends-on "init-loader")
```

init.elのADDは、PalletのREADME.mdにあるようにCaskをインストールしてから行います。

``` el ~/docker_apps/emacs/.emacs.d/init.el
(require 'cask "~/.cask/cask.el")
(cask-initialize)
(require 'pallet)

(require 'init-loader)
(setq init-loader-show-log-after-init nil)
(init-loader-load "~/.emacs.d/inits")
```

### init-loader

init-loaderでロードするキーバインドの設定です。

``` el ~/docker_apps/emacs/.emacs.d/inits/00-keybinds.el
(define-key global-map "\C-h" 'delete-backward-char)
(define-key global-map "\M-?" 'help-for-help)
```

init-loaderでロードするファイル系の設定です。

``` el ~/docker_apps/emacs/.emacs.d/inits/01-files.el
(setq backup-inhibited t)
(setq next-line-add-newlines nil)
(setq-default tab-width 4 indent-tabs-mode nil)
```

### Dockerfile

Dockerfileは日本語化とよく使う開発用のパッケージをインストールしています。

``` bash ~/docker_apps/emacs/Dockerfile
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
                       emacs24-nox emacs24-el git byobu wget curl unzip tree \
                       python

WORKDIR /root

# Cask
RUN curl -fsSkL https://raw.github.com/cask/cask/master/go | python
ADD .emacs.d /root/.emacs.d/
RUN /bin/bash -c 'echo export PATH="/root/.cask/bin:$PATH" >> /root/.profile' && \
    /bin/bash -c 'source /root/.profile && cd /root/.emacs.d && cask install'

# Clean
RUN apt-get clean && rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*
```


### build && run

``` bash
$ cd ~/docker_apps/emacs
$ docker build -t masato/emacs .
$ docker run -it --rm masato/emacs bash
```

### いままで通りpackage.elが使える

`M-x package-install`でパッケージをインストールしても、Caskファイルにインストールした内容を追加してくれます。
いままで通りpackage.elが使えるので便利です。

```
M-x package-install clojure-mode
```

自動的にCaskファイルに追記されます。

``` bash ~/docker_apps/emacs/Cask
(source melpa)

(depends-on "clojure-mode")
(depends-on "pallet")
```

