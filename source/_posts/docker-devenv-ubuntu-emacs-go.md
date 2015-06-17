title: 'Docker開発環境 - Ubuntu14.04 x Go1.2.1 x Emacs24.3'
date: 2014-05-29 22:15:39
tags:
 - Dockerfile
 - DockerDevEnv
 - Go
 - Emacs
description: vagrant-lxcにGo開発環境を作ったときのgvmはなぜかDockerコンテナにインストールできませんでした。rbenvは仕方なくでしたが、Dockerコンテナ内で言語のバージョン管理が不要な気がするので、apt-getでgolangをインストールします。また、Emacsのgo-modeもpackage.elでインストールします。GOROOTとGOPATHはどうすればよいか悩んでいたところ、Single GOPATHを読んで参考にしました。
---

[vagrant-lxc](/2014/05/16/vagrant-lxc-idcf-go/)にGo開発環境を作ったときの[gvm](https://github.com/moovweb/gvm)はなぜかDockerコンテナにインストールできませんでした。
rbenvは仕方なくでしたが、Dockerコンテナ内で言語のバージョン管理が不要な気がするので、
apt-getでgolangをインストールします。また、Emacsのgo-modeもpackage.elでインストールします。

GOROOTとGOPATHはどうすればよいか悩んでいたところ、[Single GOPATH](http://bsilverstrim.blogspot.jp/2014/03/use-single-gopath-in-go-golang.html)を読んで参考にしました。

<!-- more -->

### Dockerfile

Ubuntu14.04のapt-getでGo1.2.1がインストールされました。2014-05-29では1.2.2がstable、1.2.1は新しいほうです。
Dockerfileの環境変数の設定がいまひとつ分かないので、ENVで設定しました。
Goのインストールを、昨日までのDockerfileに追加します。

``` bash ~/docker_apps/phusion/Dockerfile
## Go
RUN apt-get install -y golang
RUN mkdir -p /root/gocode/src /root/gocode/bin /root/gocode/pkg
ENV GOPATH /root/gocode
```

### init-loader.elとgo-mode
DockerホストのEmacsでgo-modeを`M-x list-packages`からしてインストールします。
ELPAのパッケージをイメージにADDするため、作業ディレクトリのdotfilesにコピーします。

``` bash
# cd ~/docker_apps/phusion
# cp -R ~/.emacs.d/elpa/go-mode-20140409.928 ./dotfiles/.emacs.d/elpa/
# cp ~/.emacs.d/inits/06-go-mode.el ./dotfiles/.emacs.d/elpa/inits
```

dotfiles/initsディレクトリはこんな感じになりました。

``` bash
$ tree -a dotfiles/.emacs.d/inits
dotfiles/.emacs.d/inits
├── 00-keybinds.el
├── 02-files.el
├── 03-multi-term.el
├── 04-web-mode.el
├── 05-color-theme-solarized.el
├── 06-js3-mode.el
└── 07-go-mode.el
```

02-files.elでソフトタブ 4にしています。

``` el ~/docker_apps/phusion/dotfiles/.emacs.d/inits/02-files.el
(setq backup-inhibited t)
(setq next-line-add-newlines nil)
(setq-default tab-width 4 indent-tabs-mode nil)
```

Goのプログラムを書く場合、PythonやRubyと違うのでgo-mode設定をフックさせます。

* ソフトタブは使わない
* 4タブにする
* 保存の前にgofmtする

``` el ~/docker_apps/phusion/dotfiles/.emacs.d/inits/07-go-mode.el
(add-hook 'go-mode-hook
          (lambda ()
            (add-hook 'before-save-hook 'gofmt-before-save)
            (setq tab-width 4)
            (setq indent-tabs-mode 1)))
```

### docker build & run

イメージの作成は、今回からタグでバージョン管理するようにしました。タグをつけると不要なイメージの削除が簡単になります。
コンテナは使い捨てで増えていくので、タグからどのバージョンで起動したか分かると便利です。

``` bash
$ docker build -t masato/baseimage:1.0 .
$ docker run -t -i --rm  masato/baseimage:1.0 /sbin/my_init /bin/bash
root@e9bcb047dca7:/#
```

### Goの確認

DockerfileでインストールしたGoの確認をします。GOROOTとGOPATHが設定されました。
これからは`$GOPATH/src`に下に自分のプロジェクトを書いていきます。

``` bash
# which go
/usr/bin/go
# go version
go version go1.2.1 linux/amd64
# go env GOROOT
/usr/lib/go
# go env GOPATH
/root/gocode
# ls $GOPATH
bin  pkg  src
``` 

### まとめ

ようやくGoの環境ができました。フレームワークはどれで書こうかまだ決めかねています。
Martiniが`Go Way`でないとか、injectionが分かりずらいとか、`Rails Way`や`DRY論`疲れていると、
何となく分かる気がします。
