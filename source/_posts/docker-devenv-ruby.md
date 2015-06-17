title: 'Dockerで開発環境をつくる - Ruby2.0編'
date: 2014-05-23 22:10:22
tags:
 - Dockerfile
 - DockerDevEnv
 - Ruby
 - Ubuntu
description: Sinatraで書かれているDashingを使ったプログラムを書くことになったので、昨日作ったDockerfileを修正しながらRuby2.0の開発環境をつくります。今までrbenvとruby-buildで環境を作っていたのですが、Rakeをcrontabで使う場合やSSHでコマンド実行する場合面倒でした。Docker環境なのでシステムワイドにapt-getでRuby 2.0をインストールしたかったのですが
---

Sinatraで書かれている[Dashing](http://shopify.github.io/dashing/)を使ったプログラムを書くことになったので、
[昨日](/2014/05/22/docker-devenv-rebuild/)作ったDockerfileを修正しながらRuby2.0の開発環境をつくります。

今までrbenvとruby-buildで環境を作っていたのですが、Rakeをcrontabで使う場合やSSHでコマンド実行する場合面倒でした。
Docker環境なのでシステムワイドにapt-getでRuby 2.0をインストールしたかったのですが、、

### TL;DR

rbenvに戻しました。Ubuntuのパッケージ管理の依存関係がまだよくわかりません。

<!-- more -->


### Dockerfile

昨日作ったDockerfileに追加した内容

* timezoneをAsia/Tokyoにする
* aptのミラーを日本にする
* Ruby2.0のインストールをする
* dotfiles/.emacs.dを予めinit-loader, multi-term, enh-ruby-modeを設定済み
* dotfiles/.vimrcに、バックアップしない設定済み

``` Dokerfile

FROM phusion/baseimage:0.9.10

# Set correct environment variables.
ENV HOME /root

# Regenerate SSH host keys.
RUN /etc/my_init.d/00_regen_ssh_host_keys.sh

## Install an SSH of your choice.
ADD google_compute_engine.pub /tmp/your_key
RUN cat /tmp/your_key >> /root/.ssh/authorized_keys && rm -f /tmp/your_key

## apt-get update
RUN sed -i~ -e 's/archive.ubuntu.com/ftp.jaist.ac.jp/' /etc/apt/sources.list \
 && apt-get update

## Japanese Environment
RUN apt-get install -y language-pack-ja
ENV LANG ja_JP.UTF-8
RUN update-locale LANG=ja_JP.UTF-8
RUN mv /etc/localtime /etc/localtime.orig
RUN ln -s /usr/share/zoneinfo/Asia/Tokyo /etc/localtime

## Development Environment
RUN apt-get install -y emacs24-nox emacs24-el git

## Ruby
RUN apt-get install -y ruby2.0 ruby2.0-dev
RUN update-alternatives --install /usr/bin/ruby ruby /usr/bin/ruby2.0 10
RUN gem install bundler rubygems-update --no-rdoc --no-ri
RUN update_rubygems

## dotfiles
ADD dotfiles /root/

CMD ["/sbin/my_init"]

RUN apt-get clean && rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*
```

web-modeの設定

``` el ~/.emacs.d/inits/04-web-mode.el
(require 'web-mode)
(add-to-list 'auto-mode-alist '("\\.phtml\\'" . web-mode))
(add-to-list 'auto-mode-alist '("\\.tpl\\.php\\'" . web-mode))
(add-to-list 'auto-mode-alist '("\\.[gj]sp\\'" . web-mode))
(add-to-list 'auto-mode-alist '("\\.as[cp]x\\'" . web-mode))
(add-to-list 'auto-mode-alist '("\\.erb\\'" . web-mode))
(add-to-list 'auto-mode-alist '("\\.mustache\\'" . web-mode))
(add-to-list 'auto-mode-alist '("\\.djhtml\\'" . web-mode))
(add-to-list 'auto-mode-alist '("\\.html?\\'" . web-mode))
```

multi-termの設定
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

;;multi-term
(setq load-path(cons "~/.emacs.d/" load-path))
(setq multi-term-program "/bin/bash")
(require 'multi-term)
```


### イメージの作成

Dockerfileからビルドしてイメージを作成します。

``` bash
$ docker build -no-cache -t masato/baseimage .
```

コンテナを作成して、インストールしたRubyとGemのバージョンを確認します。

``` bash
$ docker run -t -i --rm  masato/baseimage /sbin/my_init /bin/bash
*** Running /etc/my_init.d/00_regen_ssh_host_keys.sh...
*** Running /etc/rc.local...
*** Booting runit daemon...
*** Runit started as PID 9
*** Running /bin/bash...
root@25179d090ff5:/# ruby -v
ruby 2.0.0p384 (2014-01-12) [x86_64-linux-gnu]
root@25179d090ff5:/# gem -v
2.2.2
```


### rbenvで作り直し

apt-getでインストールしたRuby 2.0は、Dashingのインストールでエラーが出てしまいました。

``require': no such file to load — mkmf (LoadError)`

Ruby1.9だと問題ないです。

``` bash
apt-get install -y ruby1.9.3 ruby-dev
gem install bundler --no-rdoc --no-ri
```

どうもUbuntu場合、Rubyパッケージの依存関係があるようで、rbenvを使って作るようにします。
apt-getでインストールしたかったのですが、やはりRubyはrbenvしないとだめみたいです。
2.1.2をシステムワイドにインストールします。

Dockerfileを編集してrbenvからインストールするようにします。

``` bash ~/docker_apps/phusion/Dockerfile
## Ruby 2.1.2 rbenv install
RUN apt-get install -y build-essential
RUN apt-get install -y zlib1g-dev libssl-dev libreadline-dev libyaml-dev libxml2-dev libxslt-dev
RUN apt-get install -y sqlite3 libsqlite3-dev

RUN git clone https://github.com/sstephenson/ruby-build.git .ruby-build
RUN .ruby-build/install.sh
RUN rm -fr .ruby-build
RUN ruby-build 2.1.2 /usr/local
RUN gem update --system
RUN gem install bundler --no-rdoc --no-ri
```

コンテナを起動してバージョンを確認します。

``` bash
$ docker run -t -i --rm  masato/baseimage /sbin/my_init /bin/bash*** Running /etc/my_init.d/00_regen_ssh_host_keys.sh...
*** Running /etc/rc.local...
*** Booting runit daemon...
*** Runit started as PID 13
*** Running /bin/bash...
root@967fb55b5535:/# ruby -v
ruby 2.1.2p95 (2014-05-08 revision 45877) [x86_64-linux]
root@967fb55b5535:/# gem -v
2.2.2
root@967fb55b5535:/# which ruby
/usr/local/bin/ruby
root@967fb55b5535:/# which gem
/usr/local/bin/gem
```

### まとめ

まだ試行錯誤中なのですが、Ruby2.0用の開発環境をつくるDockerfileができました。
なるべくパッケージを使いたかったのですが、今回はrbenvしています。
次は新しいイメージをつかって、Dashingのアプリを開発します。


