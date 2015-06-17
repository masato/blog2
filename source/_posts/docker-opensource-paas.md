title: "DockerでオープンソースPaaS - Part1: はじめに"
date: 2014-07-02 20:35:29
tags:
 - まとめ
 - Docker
 - DockerPaaS
 - HTTPRouting
 - tsuru
 - Yandex
 - Deis
 - Shipyard
description: Dockerをバックエンドに使うオープンソースのPaaSも増えてきて、HerokuのBuildpackを使えるようになったり、CloudFoundryのバックエンドにDockerを使えるようになりました。Deimosの前身Docker-on-Mesosから入ったので、さっぱり意味がわかりませんでした。本当はPaaSって何？で定義が難しいのですが、Dockerエコシステムの拡がりでいろいろな挑戦をしているサービスやライブラリが増えています。自分のやりたいことベースで少しずつ整理していきたいと思います。私の用途だとFlynnやDeisでも規模が大きい感じです。
---

* `Update 2014-08-14`: [Deis in IDCFクラウド - Part1: CoreOSクラスタとDeisインストール](/2014/08/14/deis-in-idcf-cloud-install/)

Dockerをバックエンドに使うオープンソースのPaaSも増えてきて、
HerokuのBuildpackを使えるようになったり、CloudFoundryのバックエンドにDockerを使えるようになりました。

Deimosの前身Docker-on-Mesosから入ったので、さっぱり意味がわかりませんでした。
本当はPaaSって何？で定義が難しいのですが、Dockerエコシステムの拡がりでいろいろな挑戦をしているサービスやライブラリが増えています。自分のやりたいことベースで少しずつ整理していきたいと思います。

私の用途だとFlynnやDeisでも規模が大きい感じです。

* Dockerに慣れてきたので、どんどんアプリを書いて動かしたい
* Buildpackを使って、Dockerfileから言語の実行環境を分離したい
* データベースをDockerから分離してサービスとして使いたい
* IaaS上で、カジュアル && ミニマルにPaaSを構築したい
* Herokuみたいな[HTTP Routing](https://devcenter.heroku.com/articles/http-routing)がしたい

そうすると、Dokkuかtsuruが候補にあがります。

DokkuとFlynnは、`Jeff Lindsay`さんが開発しています。
PistonでBOSHの`OpenStack Provider`を書いていた人なので、最近の流れを作った一人だと思います。

一方のtsuruは、ブラジル最大のメディア複合企業であるglobo.comが開発しています。
[globo.com s2 python](https://speakerdeck.com/andrewsmedina/globo-dot-com-s2-python)のスライドが、`loves Python`なのと、西海岸以外のオープンソースもウォッチしたいので、これからtsuruをしばらく触ってみようと思います。

Dockerのエコシステムでは、中国のBaidu、ロシアのYandexもあまり情報がありませんが、Dockerを採用しています。

<!-- more -->
 
### BuildpackをDockerで使うためのツール

* [Buildstep](https://github.com/progrium/buildstep)
* [building](https://github.com/CenturyLinkLabs/building)

### 社内向け開発環境

* [Dokku](https://github.com/progrium/dokku)
* [tsuru](https://github.com/tsuru/tsuru)
* [Octohost](https://github.com/octohost/octohost)

### エンタープライズ向け分散開発環境

* [Flynn](https://github.com/flynn/flynn)
* [Deis](https://github.com/deis/deis)

### Dockerコンテナ管理

* [Panamax](http://panamax.io/)
* [Fig](http://orchardup.github.io/fig/)
* [Shipyard](https://github.com/shipyard/shipyard)
* [Helios](https://github.com/spotify/helios)
* [Centurion](https://github.com/newrelic/centurion)
* [decking](https://github.com/makeusabrew/decking)

### Dockerクラスタ管理 (Datacenter-as-a-Computer)

* [fleet](https://github.com/coreos/fleet)
* [Docker BOSH Release](https://github.com/cf-platform-eng/docker-boshrelease)
* [Deimos](https://github.com/mesosphere/deimos)
* [Kubernetes](https://github.com/GoogleCloudPlatform/kubernetes)

### HTTP Routing

* [Hipache](https://github.com/dotcloud/hipache)
* [OpenResty](https://github.com/openresty/ngx_openresty)
* [confd](https://github.com/kelseyhightower/confd)
* [ngrok](https://github.com/inconshreveable/ngrok)
* [Vulcand](https://github.com/mailgun/vulcand)
* [Havok](https://github.com/ehazlett/docker-havok)
 