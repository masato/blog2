title: 'Dockerで開発環境をつくる - Node.js 0.10.25編'
date: 2014-05-24 01:39:07
tags:
 - Dockerfile
 - DockerDevEnv
 - Nodejs
 - IDCFオブジェクトストレージ
 - Ubuntu
description: ここにたくさんリストされているStatic Site Generatorを使うと、無料でクラウドサービスにホスティングできるのが魅力です。このサイトはGitHub Pages上ですが、AmazonS3へWebサイトをホスティングすることもできます。Hexoでは、knoxを使いAmazonS3 APIを操作するため、最初にNode.jsの環境を作ります。
---

[ここ](http://staticsitegenerators.net/)にたくさんリストされている`Static Site Generator`を使うと、無料でクラウドサービスにホスティングできるのが魅力です。
このサイトは[GitHub Pages](https://pages.github.com/)上ですが、AmazonS3へWebサイトを[ホスティング](http://docs.aws.amazon.com/AmazonS3/latest/dev/WebsiteHosting.html)することもできます。

Hexoでは、[knox](https://github.com/LearnBoost/knox)を使い`AmazonS3 API`を操作するため、最初にNode.jsの環境を作ります。


<!-- more -->

昨日Ruby 2.0まで設定を行ったDockerfileに、Node.jsのインストールを追加します。

### Dockerfile


今回Dockerfileに追加した内容。

* Node.jsとnpmパッケージマネージャー
* byobuのマルチプレクサ
* [js3-mode](https://github.com/thomblake/js3-mode)
* [color-theme-solarized](https://github.com/sellout/emacs-color-theme-solarized)

### byotuと日本語環境

なぜかbyobuを起動すると、かってに毎秒スクロールしてしまいます。
理由はまだわかりませんが、とりあえず`byobu-config`でステータス通知の設定をやり直すと解消します。

``` Dokerfile
## Development Environment
RUN apt-get install -y emacs24-nox emacs24-el git byobu wget curl

## Node.js
RUN apt-get install -y nodejs npm
ENV NODE_PATH /usr/local/lib/node_modules
RUN ln -s /usr/bin/nodejs /usr/bin/node
```

コンテナを作成して、インストールしたRubyのバージョンを確認します。

``` bash
$ docker run -t -i --rm  masato/baseimage /sbin/my_init /bin/bash
*** Running /etc/my_init.d/00_regen_ssh_host_keys.sh...
*** Running /etc/rc.local...
*** Booting runit daemon...
*** Runit started as PID 14
*** Running /bin/bash...
root@2a2a53690e4b:/# nodejs -v
v0.10.25
root@2a2a53690e4b:/# npm -v
1.3.10
```

nodeのシムリンクを確認します。

``` bash
root@2a2a53690e4b:/# which node
/usr/bin/node
```

### まとめ

次回はknoxをインストールして、IDCFオブジェクトストレージのAPIを試しながらホスティングの環境を用意していきます。