title: "ES6で書くIsomorphicアプリ入門 - Part2: ブラウザから使えるREPL"
date: 2015-07-03 14:25:59
tags:
 - ES6
 - Isomorphic
 - REPL
 - iojs
 - Cloud9
 - Ambly
description: ES6で書くIsomorphicアプリ開発はまだGetting Started以前の知識しかありません。道のりは長そうですが、次世代の開発の準備は少しずつしておきたいです。ClojureScriptにはiOSで動くAmblyというREPLもあります。気になったときにターミナルを開かなくてもどこでもREPLが使えるようになればとても便利です。電卓の代わりにもなります。
---

ES6で書くIsomorphicアプリ開発はまだGetting Started以前の知識しかありません。道のりは長そうですが、次世代の開発準備は少しずつしておきたいです。ClojureScriptにはiOSで動くAmblyというREPLもあります。気になったときにターミナルを開かなくてもどこでもREPLが使えるようになれば、電卓の代わりにもなります。

<!-- more -->

[Getting Started with ES6 – The Next Version of JavaScript](http://weblogs.asp.net/dwahlin/getting-started-with-es6-%E2%80%93-the-next-version-of-javascript)のポストにブラウザからオンラインで使えるES6のREPLがいくつか紹介されています。

## オンラインブラウザ

[Traceur](https://google.github.io/traceur-compiler)や[Babel](https://babeljs.io/)のコンパイラのホームページにブラウザから使えるREPLが公開されています。インストールが不要なのでちょっと試す場合に一番お手軽です。

* [ES6 Fiddle](http://www.es6fiddle.com/)
* [Traceur Transcoding Demo](https://google.github.io/traceur-compiler/demo/repl.html#)
* [Babel REPL](https://babeljs.io/repl/)
* [TypeScript Playground](http://www.typescriptlang.org/Playground)


## Chrome Extension

[Scratch JS](https://github.com/richgilbank/Scratch-JS)はChrome Extensionです。開発者のコメントにもありますが、誰かのブログを読んでいて試したいコードがあったときに、`option + command + I`を押してDeveloper Tools からすぐに確認できます。

* [Scratch JS Chrome Extension](https://chrome.google.com/webstore/detail/scratch-js/alploljligeomonipppgaahpkenfnfkn)


## DockerとCloud9

(https://registry.hub.docker.com/u/library/iojs/)イメージを使います。現在の最新バージョンの2.3です。[io.jsにおけるES6](https://iojs.org/ja/es6.html)にフラグについて書いてあるように、アロー関数はまだデフォルトで対応されていません。harmonyフラグをつけて起動する必要があります。

```bash
$ docker run --rm -it iojs:2.3 iojs --harmony_arrow_functions
```

私の場合[io.js](https://registry.hub.docker.com/u/library/iojs/)をベースイメージにして[Cloud9をクラウド上](/2015/06/08/cloud9-on-idcf-install/)にブラウザから使える開発環境を作っています。

```bash Dockerfile
FROM iojs:2.3
MAINTAINER Masato Shimizu <ma6ato@gmail.com>

RUN apt-get update && apt-get install -y vim

RUN git clone https://github.com/c9/core.git /cloud9 && \
    cd /cloud9 && ./scripts/install-sdk.sh

RUN npm install hexo-cli -g

RUN wget -O - https://storage.googleapis.com/golang/go1.4.2.linux-amd64.tar.gz | tar -xzC /usr/local -f - && \
    echo "export GOPATH=/workspace/gocode" >> /root/.profile && \
    echo "export PATH=$PATH:/usr/local/go/bin:/workspace/gocode/bin" >> /root/.profile

WORKDIR /workspace

RUN apt-get clean && rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*
```

Cloud9のターミナルからES6のREPLが実行できます。

![cloud9-es6-repl.png](/2015/07/03/iojs-es6-isomorphic-repl/cloud9-es6-repl.png)