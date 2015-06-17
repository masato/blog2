title: "Reagent入門 - Part1: コンポーネントのモックを作る"
date: 2015-05-22 00:08:22
tags:
 - Reagent
 - ClojureScript
 - Figwheel
 - React
description: ReactをClojureScriptで書くため、Reagentの勉強をまずはratomを使わない簡単なコンポーネントを作るところからはじめようと思います。ClojureScriptとFigwheelのシンプルなテンプレートの[reagent-figwheel](https://github.com/gadfly361/reagent-figwheel)を使ったサンプルを探していると、[Static Mock Using Reagent](http://www.okaythree.com/articles/2015/04/21/static-mock-using-reagent/)のよい例が見つかりました。参考になるプログラムを読みながら実際に手を動かしながら勉強していくのが一番よいです。
---

ReactをClojureScriptで書くため、[Reagent](https://github.com/reagent-project/reagent)の勉強をまずはratomを使わない簡単なコンポーネントを作るところからはじめようと思います。ClojureScriptとFigwheelのシンプルなテンプレートの[reagent-figwheel](https://github.com/gadfly361/reagent-figwheel)を使ったサンプルを探していると、[Static Mock Using Reagent](http://www.okaythree.com/articles/2015/04/21/static-mock-using-reagent/)のよい例が見つかりました。参考になるプログラムを読みながら実際に手を動かしながら勉強していくのが一番よいです。

<!-- more -->

## プロジェクトの作成

Clojureの開発はDockerとDocker Composeを使って行います。

``` bash
$ cd ~/clojure_apps
$ tree -L 1
.
├── Dockerfile
└── docker-compose.yml
```

オフィシャルのClojureの[ベースイメージ](https://registry.hub.docker.com/_/clojure/)を使います。

```bash ~/clojure_apps/Dockerfile
FROM clojure
MAINTAINER Masato Shimizu <ma6ato@gmail.com>

WORKDIR /usr/src/app

RUN apt-get update && apt-get install sudo net-tools && \
  rm -rf /var/lib/apt/lists/*

RUN adduser --disabled-password --gecos '' --uid 1000 docker && \
  adduser docker sudo && \
  echo 'docker ALL=(ALL) NOPASSWD:ALL' >> /etc/sudoers && \
  mkdir /home/docker/.m2 && \
  chown -R docker:docker /usr/src/app /home/docker/.m2

VOLUME /home/docker/.m2
USER docker
RUN lein

ENTRYPOINT ["lein"]
```

ローカルにClojureのイメージをビルドします。

```bash
$ docker-compose build
```

カレントディレクトリをマウントするため、docker-compose.ymlは`lein new`する前は`working_dir`をコメントアウトしておきます。

```yaml ~/clojure_apps/docker-compose
lein: &defaults
  image: clojure
  volumes:
    - .:/usr/src/app
    - ./m2:/home/docker/.m2
#  working_dir: /usr/src/app/my-checklist
  ports:
    - "3449:3449"
    - "10555:10555"
```

Leiningenで[reagent-figwheel](https://github.com/gadfly361/reagent-figwheel)のテンプレートを使ったプロジェクトを作成します。

``` bash
$ docker-compose run --rm --service-ports lein new reagent-figwheel my-checklist
```

このテンプレートでは以下のようなシンプルな構成のファイルが生成されます。

``` bash
$ tree -L 3
.
├── README.md
├── dev
│   ├── user.clj
│   └── user.cljs
├── project.clj
├── resources
│   └── index.html
└── src
    └── my_checklist
        └── core.cljs
```

docker-compose.ymlの`working_dir`をアンコメントします。

```yaml ~/clojure_apps/docker-compose
lein: &defaults
  image: clojure
  volumes:
    - .:/usr/src/app
    - ./m2:/home/docker/.m2
  working_dir: /usr/src/app/my-checklist
  ports:
    - "3449:3449"
    - "10555:10555"
```

## 開発サーバーの起動

開発用のDockerホストはクラウド上にあるため、[Figwheel](https://github.com/bhauman/lein-figwheel)はリモートからパブリックIPアドレスで接続できるようにします。

```clj ~/clojure_apps/my-checklist/dev/user.cljs
(ns cljs.user
  (:require [my-checklist.core :as core]
            [figwheel.client :as figwheel :include-macros true]))

(enable-console-print!)

(figwheel/watch-and-reload
  :websocket-url "ws://210.xxx.xxx.xxx:3449/figwheel-ws"
  :jsload-callback (fn [] (core/main)))

(core/main)
```

REPLを起動します。

``` bash
$ docker-compose run --rm --service-ports lein repl
nREPL server started on port 51843 on host 127.0.0.1 - nrepl://127.0.0.1:51843
REPL-y 0.3.5, nREPL 0.2.6
Clojure 1.6.0
OpenJDK 64-Bit Server VM 1.7.0_79-b14
    Docs: (doc function-name-here)
          (find-doc "part-of-name-here")
  Source: (source function-name-here)
 Javadoc: (javadoc java-object-or-class-here)
    Exit: Control+D or (exit) or (quit)
 Results: Stored in vars *1, *2, *3, an exception in *e

user=>
```

REPLで(run)を実行してアプリの起動

``` bash
user=> (run)
2015-05-21 14:56:17.542:INFO:oejs.Server:jetty-7.6.13.v20130916
2015-05-21 14:56:17.582:INFO:oejs.AbstractConnector:Started SelectChannelConnector@0.0.0.0:10555
Starting web server on port 10555 .
#<Server org.eclipse.jetty.server.Server@7ef1accd>
user=>
```

次に(start-figwheel)を実行してFigwheelを起動します。

``` bash
user=> (start-figwheel)
Starting figwheel.
#<core$future_call$reify__6320@17e02401: :pending>
user=> Figwheel: focusing on build-id 'app'
Compiling ClojureScript.
Figwheel: Starting server at http://localhost:3449
Figwheel: Serving files from '(dev-resources|resources)/public'
Compiling "resources/public/js/app.js" from ["src" "dev"]...
Successfully compiled "resources/public/js/app.js" in 23.719 seconds.
notifying browser that file changed:  /js/app.js
notifying browser that file changed:  /js/out/goog/deps.js
notifying browser that file changed:  /js/out/my_checklist/core.js
notifying browser that file changed:  /js/out/cljs/user.js
Figwheel: client disconnected  :normal
```

Chromeブラウザを開きDockerホストのパブリックIPアドレスに接続します。

http://210.xxx.xxx.xxx:10555/

画面には以下のようなコメントが表示されます。

```
Hello, what is your name? FIXME
```

Chromeブラウザのデベロッパーツールを開くとFigwheelの起動が確認できます。

```
Figwheel: trying to open cljs reload socket
utils.cljs:31 Figwheel: socket connection established
```

## コンポーネントの作成

今回のメインテーマであるコンポーネントを作成していきます。project.cljのdependenciesディレクティブを確認するとパッケージの数は少なくシンプルな構成なのでわかりやすいです。

```clj ~/clojure_apps/my-checklist/project.clj
(defproject my-checklist "0.1.0-SNAPSHOT"
  :description "FIXME: write description"
  :url "http://example.com/FIXME"
  :license {:name "Eclipse Public License"
            :url "http://www.eclipse.org/legal/epl-v10.html"}

  :source-paths ["src" "dev"]

  :dependencies [[org.clojure/clojure "1.6.0"]
                 [org.clojure/clojurescript "0.0-3058" :scope "provided"]
                 [org.clojure/core.async "0.1.346.0-17112a-alpha"]
                 [ring "1.3.1"]
                 [compojure "1.2.0"]
                 [figwheel "0.2.5"]
                 [environ "1.0.0"]
                 [leiningen "2.5.0"]
                 [reagent "0.5.0"]]

  :min-lein-version "2.5.0"

  :plugins [[lein-cljsbuild "1.0.4"]
            [lein-environ "1.0.0"]
            [lein-figwheel "0.2.0-SNAPSHOT"]]

  :figwheel {:http-server-root "public"
             :port 3449}

  :cljsbuild {:builds {:app {:source-paths ["src" "dev"]
                             :compiler {:output-to "resources/public/js/app.js"
                                        :output-dir "resources/public/js/out"
                                        :source-map    "resources/public/js/out.js.map"
                                        :optimizations :none}}}}
  )
```

core.cljsにコンポーネントを書いていきます。フォームとリストの簡単な構成です。[Static Mock Using Reagent](http://www.okaythree.com/articles/2015/04/21/static-mock-using-reagent/)を参考にしながら`li`のリスト項目を作成する共通関数を用意してみました。

コンポーネントは一つ一つをベクターを使って書いていきます。HTMLの書き方は[Hiccup](https://github.com/weavejester/hiccup)のDSLのように直感的でわかりやすいです。リストを作成するところはベクターの中でforループがそのまま使えます。

```clj ~/clojure_apps/my-checklist/src/my_checklist/core.cljs
(ns my-checklist.core
  (:require [reagent.core :as reagent :refer [atom]]))

(defn checklist-title []
  [:h1 "朝にやること"])

(defn checkbox-input [item]
  [:li
   [:input {:type "checkbox"} (:label item)]])

(defn checklist-items []
  (let [items [{:label "水を入れる"}
               {:label "キューリグをセット"}
               {:label "コーヒーを入れる"}]]
    [:ul
     (for [item items]
       [checkbox-input item])]))

(defn add-form []
  [:form
   [:p
    [:input {:text "text" :name "task"}]]
   [checklist-items]])

(defn reset-button []
  [:button {:type "button"} "リセットする"])

(defn full-checklist []
  [:div
   [checklist-title]
   [add-form]
   [reset-button]])
   
(defn page []
  [:div
   [full-checklist]])

(defn main []
  (reagent/render-component [page] (.getElementById js/document "app")))
```

静的なコンポーネントなのでリセットボタンを押しても何も動きません。チェックボックスもrtomを使っていないのでチェックするだけの状態です。とりあえずHTMLがレンダリングできるところまで確認しました。

![reagent-static.png](/2015/05/22/reagent-tutorials-static-mock/reagent-static.png)

## コンポーネントはデータとして扱う

[BUILDING SINGLE PAGE APPS WITH REAGENT](http://yogthos.net/posts/2014-07-15-Building-Single-Page-Apps-with-Reagent.html)によい例があります。text-inputコンポーネントにrowコンポーネントネストする例です。rowコンポーネントは関数として直接実行せずデータとして定義するとReagentが必要なときに評価して実行してくれます。

```clj
(defn row [label & body]
  [:div.row
   [:div.col-md-2 [:span label]]
   [:div.col-md-3 body]])

(defn text-input [label]
  [row label [:input {:type "text" :class "form-control"}]])
```
