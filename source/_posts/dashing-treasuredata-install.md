title: 'DashingとTreasure Data - Part1: インストール編'
date: 2014-05-27 00:30:03
tags:
 - Dashing
 - AnalyticSandbox
 - TreasureData
 - DockerEnv
 - rbenv
description: RubyのDocker開発環境を使って、Dashingアプリをつくってみます。Data VisualizationのJavaScriptの中では、SquareのCubism.jsや、最近はやっているGrafanaと気に入っています。メトリクスはGraphite用のダッシュボードで表現することが多いですが、ApacheやTomcatのログをTreasure Dataに貯めているので、Counting & Timingの簡単なサンプルを作ってみようと思います。DashingはSpotifyが開発しているダッシュボードです。おなじくSpotifyが開発しているBatman.jsをSPAのライブラリに使っています。そういえばSpine.jsやSroutCoreなどとくらべて、どれにしようか昔悩んでいました。
---
[RubyのDocker開発環境](/2014/05/23/docker-devenv-ruby/)を使って、[Dashing](https://github.com/Shopify/dashing)アプリをつくってみます。
`Data Visualization`のJavaScriptの中では、[Square](https://squareup.com/)の[Cubism.js](http://square.github.io/cubism/)や、最近はやっている[Grafana](http://grafana.org/)と気に入っています。

メトリクスはGraphite用のダッシュボードで表現することが多いですが、ApacheやTomcatのログを`Treasure Data`に貯めているので、
`Counting & Timing`の簡単なサンプルを作ってみようと思います。

Dashingは[Spotify](https://www.spotify.com/)が開発しているダッシュボードです。おなじくSpotifyが開発している[Batman.js](https://github.com/batmanjs/batman)をSPAのライブラリに使っています。そういえばSpine.jsやSroutCoreなどとくらべて、どれにしようか昔悩んでいました。

<!-- more -->

### Dockerfileとビルド

今日までのDockerfileです。

``` bash ~/docker_apps/phusion/Dockerfile
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
RUN mv /etc/localtime /etc/localtime.org
RUN ln -s /usr/share/zoneinfo/Asia/Tokyo /etc/localtime

## Development Environment
RUN apt-get install -y emacs24-nox emacs24-el git byobu wget curl

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

## Node.js
RUN apt-get install -y nodejs npm
ENV NODE_PATH /usr/local/lib/node_modules
RUN ln -s /usr/bin/nodejs /usr/bin/node

## dotfiles
ADD dotfiles /root/

CMD ["/sbin/my_init"]

RUN apt-get clean && rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*
```

さっそくビルドします。rbenvとruby-buildしているのでイメージ作成に時間がかかるのが課題です。
もっと良い方法を探しています。

``` bash
$ docker build -no-cache -t masato/baseimage .
```

### コンテナの起動

Dockerホスト上へ、コンテナにマウントする作業ディレクトリを作成します。
コンテナから`git push`できるのですが、変更を加えるディレクトリはコンテナ外に置きたいです。

``` bash
$ mkdir -p ~/docker_apps/workspaces/dashing
```

Dockerコンテナを起動します。
DashingのSinatraはデフォルトで3030ポートを使うのでポートフォワードします。
また、先ほど作った作業ディレクトリをマウントします。

``` bash
$ docker run -t -i -p 3030:3030 -v ~/docker_apps/workspaces/dashing:/root/sinatra_apps masato/baseimage /sbin/my_init /bin/bash
```

### ssh-agent

DockerホストからコンテナにSSHする時に、ssh-agentを使います。
Dockerホストと同じVLAN上にあるGitのリモートリポジトリ(10.1.1.xx)には、
同じ秘密鍵のprivate_keyをコピーすることなく接続できるようになります。

``` bash
$ cd ~/docker_apps/phusion
$ eval `ssh-agent`
$ ssh-add ./private_key
$ ssh -A root@$(docker inspect CONTAINER_ID | jq -r '.[0] | .NetworkSettings | .IPAddress')
$ ssh mshimizu@10.1.1.xx
```

### byobu

`byobu-config`を起動して、Puttyからも使えるように設定します。
* 「ステータス通知の切り替え」 > logoをoff、ip_addressをon
* 「エスケープシーケンスの変更」 > エスケープキーをctrl-T

### Dashingのインストール

起動したコンテナにSSH接続します。

``` bash
$ cd ~/docker_apps/phusion
$ eval `ssh-agent`
$ ssh-add ./private_key
$ ssh -A root@$(docker inspect bb4678c8b924 | jq -r '.[0] | .NetworkSettings | .IPAddress')
root@bb4678c8b924:~#
```

dashingのインストールをします。apt-getしたruby2.0だとmkmfが見つからないエラーに
悩まされました。やはりrbenvするのが基本でした。

``` bash
# gem install dashing --no-ri --no-rdoc
```

### Dashingの起動

コンテナにマウントしたディレクトリに、Dashingアプリのひな形を作成します。
``` bash
# cd ~/sinatra_apps
# dashing new dashing_td
# cd dashing_td
```

Gemfileを編集します。twitterはコメントアウトします。
``` ruby  ~/sinatra_apps/dashing_td/Gemfile
#gem 'twitter', '>= 5.0.0'
gem 'execjs'
gem 'therubyracer'
```

bundle installします。
``` bash
# bundle install
```

ひな形のTwitterのジョブを削除します。
``` bash
# mv jobs/twitter.rb jobs/twitter.rb.orig
```

DashingのThinサーバーを起動します。

``` bash
# dashing start
Thin web server (v1.6.2 codename Doc Brown)
Maximum connections set to 1024
Listening on 0.0.0.0:3030, CTRL+C to stop
```

同梱されているsampleダッシュボードの確認をします。

http://{DockerホストのIPアドレス}:3030/sample

### Dashingアプリのgit commit

ここで一度プロジェクトを`git commit`します。
gitのpush.defaultはsimpleにします。

``` python ~/.gitconfig
[user]
        email = ma6ato@gmail.com
        name = Masato Shimizu
[push]
        default = simple
```

Dockerホスト -> コンテナ -> リモートリポジトリ(10.1.x.xx)へSSHの確認します。
``` bash
$ ssh-agent bash
$ ssh-add ./private_key
$ ssh -A root@172.17.0.3
$ ssh mshimizu@10.1.x.xx
```

リモートリポジトリを、リポジトリサーバーにログインして作成します。
``` bash
$ sudo mkdir -p /var/git/repos/dashing_apps/dashing_td.git
$ cd !$
$ sudo git init --bare --shared
$ sudo chgrp -R git /var/git/repos/dashing_apps/dashing_td.git
```

リモートリポジトリを追加します。
``` bash
# cd ~/sinatra_apps/dashing_td
# git remote add origin ssh://mshimizu@10.1.x.xx/var/git/repos/dashing_apps/dashing_td.git
# git config branch.master.remote origin
# git config branch.master.merge refs/heads/master
```

`.git/config`を確認します。

``` python ~/sinatra_apps/dashing_td/.git/config
[core]
	repositoryformatversion = 0
	filemode = true
	bare = false
	logallrefupdates = true
[remote "origin"]
  url = ssh://mshimizu@10.1.x.xx/var/git/repos/dashing_apps/dashing_td.git
	fetch = +refs/heads/*:refs/remotes/origin/*
[branch "master"]
	remote = origin
	merge = refs/heads/master
```

Dockerホストと同じVLANにあるリモートリポジトリへ、Dockerコンテナから`git push`できます。

``` bash
# git push origin master
```

### まとめ

今回はDashingアプリのひな形を使い、同梱されているsampleダッシュボードを表示するところまでできました。
次回は`Treasure Data`サービスに保存しているログデータを、定期的にクエリするジョブと、ダッシュボードに
表示するウィジェットを作成します。

Batman.jsのCoffeeScriptを使ったデータバインドに慣れると、自由にウィジェットを追加できるようになります。
