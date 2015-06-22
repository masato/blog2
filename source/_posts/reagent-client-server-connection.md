title: "Reagent入門 - Part3: クライアントとサーバーの通信パターン"
date: 2015-06-19 23:41:16
tags:
 - Reagent
 - Clojure
 - ClojureScript
 - WebSocket
 - Ajax
description: Reagentを使うとクライアント側で独立したSPAは実装できますが、サーバーとのデータ通信や、Reactのstateとpropsを抽象化してくれるratomの状態をサーバーとどうやって同期をするのかが問題です。クライアントだけで完結するアプリなら良いですが、多くのアプリはデータを永続化するためのバックエンドに何らかのデータベースが必要になります。StackOverflowに興味深い議論があったのでいくつか設計のパターンを調べました。Keeping Client State Up-To-Date In Reagent / Clojurescript Server push of data from Clojure to ClojureScrip
---

[Reagent](https://github.com/reagent-project/reagent)を使うとクライアント側で独立したSPAは実装できますが、サーバーとのデータ通信や、Reactのstateとpropsを抽象化してくれるratomの状態をサーバーとどうやって同期をするのかが問題です。クライアントだけで完結するアプリなら良いですが、多くのアプリはデータを永続化するためのバックエンドに何らかのデータベースが必要になります。StackOverflowに興味深い議論があったのでいくつか設計のパターンを調べました。


* [Keeping Client State Up-To-Date In Reagent / Clojurescript](http://stackoverflow.com/questions/27159921/keeping-client-state-up-to-date-in-reagent-clojurescript)
* [Server push of data from Clojure to ClojureScript](http://stackoverflow.com/questions/21823002/server-push-of-data-from-clojure-to-clojurescript)


<!-- more -->

## データベースの同期

ClojureScriptには[DataScript](https://github.com/tonsky/datascript)という[Datomic](http://www.datomic.com/)のクライアント版のようなインメモリのデータベースがあります。クライアント側にもデータベースを持つアーキテクチャには、[Couchbase Sync Gateway](https://github.com/couchbase/sync_gateway)の[Couchbase Lite](https://github.com/couchbase/couchbase-lite-ios)や[Meteor](https://www.meteor.com/)の[minimongo](https://www.meteor.com/mini-databases)があります。後者2つはクライアントのデータベースの状態はにautomagicallyな仕組みでサーバーと同期されますが、DataScriptの場合は同期の仕組みを作る必要がありそうです。

* [Chatting cats use DataScript for fun](http://tonsky.me/blog/datascript-chat/)
* [Another powered-by-DataScript example](http://tonsky.me/blog/acha-acha/)

とても便利そうなのですが、ちょっと複雑になりそうで手がでません。


## WebSocketとAjax (REST)

ちょうど[Hapi.js](http://hapijs.com/)や[actionhero.js](http://www.actionherojs.com/)でREST APIとWebSocketを一つのサーバーで提供する設計を試しています。どちらのプロトコルも用途にあわせて使い分けができるようにしたいです。

WebSocket、Ajax、core.asyncのすべてに対応するライブラリが増えています。

* [Sente](https://github.com/ptaoussanis/sente)
* [http-kit](http://www.http-kit.org/)
* [Chord](https://github.com/james-henderson/chord)


[Luminus](http://www.luminusweb.net/)はClojureScriptを使う場合にReagentを[推奨](http://www.luminusweb.net/docs/clojurescript.md#reagent)しています。Reagentからサーバへ[cljs-ajax](https://github.com/JulianBirch/cljs-ajax)を使ってAjax通信をします。おそらく多くのSPAの実装パターンがこれに該当すると思います。

* [cljs-ajax](https://github.com/JulianBirch/cljs-ajax)

REST APIを提供するライブラリはLiberatorを採用することが多いです。

* [Liberator](http://clojure-liberator.github.io/liberator/)


## core.async

まだ[core.async](https://github.com/clojure/core.async)について理解できていないのですが、ClojurScriptとClojureの間をcore.asyncで通信できるようです。

* [Chord](https://github.com/james-henderson/chord)
* [Aleph](https://github.com/ztellman/aleph)
* [jetty7-websockets-async](https://github.com/lynaghk/jetty7-websockets-async)


## RPC

[Hoplon](http://hoplon.io/)のサーバーサイドは[Castra](https://github.com/tailrecursion/castra)のRPCライブラリを使います。クライアント側から`mkremote`関数を使ってサーバー側`defrpc`関数を実行するようです。

[Comparing Reagent](http://hoplon.discoursehosting.net/t/comparing-reagent/393)にReagentとの比較があります。[Hoplon](http://hoplon.io/)はClojur/ClojureScriptを使いやすく統合するために[Boot](https://github.com/boot-clj/boot)を採用してビルドや開発環境が充実しています。
 

## フルスタック

Clojure / ClojureScriptのフルスタックフレームワークを調べてみました。Reagentに対応していない場合もあります。

* [Hoplon](http://hoplon.io/)
* [Luminus](http://www.luminusweb.net/)

* [Rente](https://github.com/enterlab/rente)
 * [Sente](https://github.com/ptaoussanis/sente): WebSocket / Ajax
 * [Reagent](https://github.com/reagent-project/reagent): React
 * [Bootstrap](http://getbootstrap.com/): CSS

* [closp](https://github.com/sveri/closp)
 * [H2](http://www.h2database.com/html/main.html): relational database
 * [http-kit](http://www.http-kit.org/): async http server
 * [Figwheel](https://github.com/bhauman/lein-figwheel) : liver reload
 * [Reagent](https://github.com/reagent-project/reagent): React
 * [DataScript](https://github.com/tonsky/datascript): client side database
 * [Bootstrap](http://getbootstrap.com/): CSS
 * [Selmer](https://github.com/yogthos/Selmer): template engine

* [clj-crud](https://github.com/thegeez/clj-crud)
 * [Ring](https://github.com/ring-clojure/ring): http server
 * [Compojure](https://github.com/weavejester/compojure): routing
 * [Liberator](http://clojure-liberator.github.io/liberator/): request flow
 * [Enlive](https://github.com/cgrand/enlive): template
 * [Quiescent](https://github.com/levand/quiescent): React
 * [cljs-ajax](https://github.com/JulianBirch/cljs-ajax)
 * [Figwheel](https://github.com/bhauman/lein-figwheel) : liver reload
 * [DataScript](https://github.com/tonsky/datascript): client side database

* [system](https://github.com/danielsz/system)
 * [H2](http://www.h2database.com/html/main.html): relational database
 * [http-kit](http://www.http-kit.org/): async http server
 * [Aleph](https://github.com/ztellman/aleph): async http server
 * [Sente](https://github.com/ptaoussanis/sente): WebSocket / Ajax
 * [Monger](http://clojuremongodb.info/): MongoDB
 * [Datomic](http://www.datomic.com/): Immutable database

* [Chestnut](https://github.com/plexus/chestnut)
 * [Figwheel](https://github.com/bhauman/lein-figwheel) : liver reload
 * [Weasel](https://github.com/tomjakubowski/weasel): REPL
 * [Om](https://github.com/omcljs/om): React
 * [Ring](https://github.com/ring-clojure/ring): http server
 * [http-kit](http://www.http-kit.org/) : async http server
 * [jetty7-websockets-async](https://github.com/lynaghk/jetty7-websockets-async): async http server


## まとめ

[Keeping Client State Up-To-Date In Reagent / Clojurescript](http://stackoverflow.com/questions/27159921/keeping-client-state-up-to-date-in-reagent-clojurescript)の議論で気にいったパターンはSPAをデータベースのクライアントの1つとみなすデザインです。


処理に必要なデータはスナップショットとして最初にクライアントに取得します。SPA側でデータを加工したあとにサーバーにsubmitします。SPAが対象のデータをリモートで処理している間の一貫性は保証されますが、処理中に他のクライアントがサーバーのデータ更新をコミットしていればsubmitに失敗して操作をやり直します。

これは昔からあるアーキテクチャでリッチクライアントと呼ばれていたFlexやGWT、ExtJSなどでよく実装していた手法です。業務系アプリだと意外とうまくいきます。同じレコードを並行して更新させないようにしたり、頻繁にデータを更新しないなど、仕様を工夫するのが現実的なようです。
