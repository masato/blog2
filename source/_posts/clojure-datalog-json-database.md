title: "ClojureでDatalogやJSON Storageを扱うためのリソース"
date: 2015-05-24 11:32:21
tags:
 - Clojure
 - Datalog
 - Datomic
 - Liberator
 - Compojure
description: Clojureの勉強をしていてそろそろデータベースを使いたくなってきました。JDCBでRDBMSに接続するのが一番簡単そうです。せっかくなのでJSON Document StorageやDatalogを選んでみようと思います。まずは情報集めからはじめます。
---

Clojureの勉強をしていてそろそろデータベースを使いたくなってきました。JDCBでRDBMSに接続するのが一番簡単そうです。せっかくなのでJSON Document StorageやDatalogを選んでみようと思います。まずは情報集めからはじめます。

<!-- more -->

## はじめに

Redditに[A database that likes clojure..](https://www.reddit.com/r/Clojure/comments/2zmqg8/a_database_that_likes_clojure/)というちょうどよい議論がありました。ClojureといえばRich Hickey氏のDatomicですがプロプライエタリです。できればオープンソースでいきたいところです。DataScriptはクライアントサイドなのでサーバーとの同期するにはどうしようか悩みます。もっともDatalogとかLog-structured StorageにこだわらなければRDBをバックエンドにしてREST APIでJSONを返すWebサービスを作ってしまった方が楽な感じです。

## Datalog

* [Datomic](http://www.datomic.com/)
* [DataScript](https://github.com/tonsky/datascript)
* [PossibleDB](https://github.com/runexec/possibledb)
* [Cascalog](https://github.com/nathanmarz/cascalog)

## JSON Storage

### ArangoDB

* [Clarango](https://github.com/edlich/clarango)

### RethinkDB

* [clj-rethinkdb](https://github.com/apa512/clj-rethinkdb)
* [Revise](https://github.com/bitemyapp/revise)

### CouchDB

* [Clutch](https://github.com/clojure-clutch/clutch)

### Redis

* [Carmine](https://github.com/ptaoussanis/carmine)

### EventStore

* [EventStore](https://github.com/EventStore/EventStore)

### ElasticSearch

* [Elastisch](http://clojureelasticsearch.info/)

### MongoDB

* [Monger](https://github.com/michaelklishin/monger)

## Building REST API Service

* [Yesql](https://github.com/krisajenkins/yesql)
* [Liberator](https://github.com/clojure-liberator/liberator)
* [Compojure](https://github.com/weavejester/compojure)
* [Compojure-api](https://github.com/metosin/compojure-api)
* [ring-json](https://github.com/ring-clojure/ring-json)
* [Ceshire](https://github.com/dakrone/cheshire)

