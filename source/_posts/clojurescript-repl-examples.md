title: "ClojureScript REPL - lein-cljsbuild, Piggieback, Austin"
date: 2014-09-25 01:03:51
tags:
 - DockerDevEnv
 - Clojure
 - ClojureScript
 - REPL
description: ClojureScriptはClojureで書いたコードをJavaScriptにコンパイルできる言語です。JavaScript VM上で動くのが特徴です。エンジンがJavaScriptのClojureサブセットなので一部使えない機能もあります。ClojureScriptのREPLをいくつか試してみましたが、Rhino上で動かしてたりClojureのREPLから起動したりと、これ自体あまり意味がありません。Reactのインタフェースに使うOmとか、WebのUIを書くReactive Programming言語として魅力がありそうですが、Polymer.dartの方が素直な感じです。
---

ClojureScriptはClojureで書いたコードをJavaScriptにコンパイルできる言語です。`JavaScript VM`上で動くのが特徴です。エンジンが`JavaScript`のClojureサブセットなので一部使えない機能もあります。

ClojureScriptのREPLをいくつか試してみましたが、Rhino上で動かしてたりClojureのREPLから起動したりと、これ自体あまり意味がありません。

[React](http://facebook.github.io/react/)のインタフェースに使う[Om](https://github.com/swannodette/om)とか、WebのUIを書く`Reactive Programming`言語として魅力がありそうですが、Polymer.dartの方が素直な感じです。

<!-- more -->

### lein-cljsbuild

leinのdefaultテンプレートでプロジェクトを作成します。

``` bash
$ lein new lein-csjbuild-example
```

project.cljに:dependenciesと、:plugins、:cljsbuildを追加します。

``` clj ~/lein-csjbuild-example/project.clj
(defproject lein-csjbuild-example "0.1.0-SNAPSHOT"
  :description "FIXME: write description"
  :url "http://example.com/FIXME"
  :license {:name "Eclipse Public License"
            :url "http://www.eclipse.org/legal/epl-v10.html"}
  :dependencies [[org.clojure/clojure "1.6.0"]
                 [org.clojure/clojurescript "0.0-2342"]]
  :plugins [[lein-cljsbuild "1.0.3"]]
  :cljsbuild {:builds [{:source-paths ["src/cljs"]
                        :compiler {:output-to "resources/public/scripts/app.js"
                                   :optimizations :whitespace
                                   :pretty-print true}}]})
```

`repl-rhino`を起動します。

``` bash
$ cd lein-csjbuild-example/
$ lein trampoline cljsbuild repl-rhino
Running Rhino-based ClojureScript REPL.
To quit, type: :cljs/quit
ClojureScript:cljs.user>
```

### mies

[mies](https://github.com/swannodette/mies)は`A minimal ClojureScript template`の略です。
このテンプレートを使うと最小構成のClojureScriptのプロジェクトを作成できます。

``` bash
$ lein new mies mies-example
```

以下のproject.cljが自動生成されます。

``` clj ~/mies-example/project.clj
(defproject mies-example "0.1.0-SNAPSHOT"
  :description "FIXME: write this!"
  :url "http://example.com/FIXME"

  :dependencies [[org.clojure/clojure "1.6.0"]
                 [org.clojure/clojurescript "0.0-2311"]]

  :plugins [[lein-cljsbuild "1.0.4-SNAPSHOT"]]

  :source-paths ["src"]

  :cljsbuild {
    :builds [{:id "mies-example"
              :source-paths ["src"]
              :compiler {
                :output-to "mies_example.js"
                :output-dir "out"
                :optimizations :none
                :source-map true}}]})
```

先ほど手動で追加した`lein-cljsbuild`の設定が予め記述されているので、このまま`repl-rhino`が起動できます。

``` bash
$ lein trampoline cljsbuild repl-rhino
Running Rhino-based ClojureScript REPL.
To quit, type: :cljs/quit
ClojureScript:cljs.user>
```

### Piggieback

[Piggieback](https://github.com/cemerick/piggieback)はClojureのREPLからClojureScriptのREPLを起動するミドルウェアです。

プロジェクトを作成します。

``` bash
$ lein new piggieback-example
```
 
project.cljに、:dependenciesと:repl-optionsを追加します。

``` clj ~/piggieback-example/project.clj
(defproject piggieback-example "0.1.0-SNAPSHOT"
  :description "FIXME: write description"
  :url "http://example.com/FIXME"
  :license {:name "Eclipse Public License"
            :url "http://www.eclipse.org/legal/epl-v10.html"}
  :dependencies [[org.clojure/clojure "1.6.0"]
                 [org.clojure/clojurescript "0.0-2342"]
                 [com.cemerick/piggieback "0.1.3"]]
  :repl-options {:nrepl-middleware [cemerick.piggieback/wrap-cljs-repl]})
```

`lein repl`でREPLを起動します。

``` bash
$ cd piggieback-example
$ lein repl

nREPL server started on port 42514 on host 127.0.0.1 - nrepl://127.0.0.1:42514
REPL-y 0.3.5, nREPL 0.2.6
Clojure 1.6.0
OpenJDK 64-Bit Server VM 1.7.0_65-b32
    Docs: (doc function-name-here)
          (find-doc "part-of-name-here")
  Source: (source function-name-here)
 Javadoc: (javadoc java-object-or-class-here)
    Exit: Control+D or (exit) or (quit)
 Results: Stored in vars *1, *2, *3, an exception in *e
user=>
```

`cljs-repl`を起動します。

``` bash
user=>  (cemerick.piggieback/cljs-repl)
Type `:cljs/quit` to stop the ClojureScript REPL
nil
cljs.user=>
```


### Austin

[Austin](https://github.com/cemerick/austin)はClojureScriptの`browser-REPL`を便利にしてくれます。
[Sample Project](https://github.com/cemerick/austin/tree/master/browser-connected-repl-sample)の手順を読みながらサンプルを実行します。

サンプルはlocalhost環境ですが、今回はDockerコンテナで実行しているのでブラウザは外部から接続する必要があります。DockerコンテナのIPアドレス確認します。

``` bash
$ docker inspect -format="&#123;&#123; .NetworkSettings.IPAddress }}" 247fbe1567c1
172.17.1.128
```

サンプルプロジェクトを`git clone`します。

``` bash
$ git clone https://github.com/cemerick/austin.git
$ cd austin/browser-connected-repl-sample
```

サンプルのClojureScriptをコンパイルして、REPLを起動します。

``` bash
$ lein do cljsbuild once, repl
Compiling ClojureScript.
Compiling "target/classes/public/app.js" from ["src/cljs"]...
Successfully compiled "target/classes/public/app.js" in 9.939 seconds.
nREPL server started on port 48980 on host 127.0.0.1 - nrepl://127.0.0.1:48980
REPL-y 0.3.5, nREPL 0.2.6
Clojure 1.5.1
OpenJDK 64-Bit Server VM 1.7.0_65-b32
    Docs: (doc function-name-here)
          (find-doc "part-of-name-here")
  Source: (source function-name-here)
 Javadoc: (javadoc java-object-or-class-here)
    Exit: Control+D or (exit) or (quit)
 Results: Stored in vars *1, *2, *3, an exception in *e

cemerick.austin.bcrepl-sample=>
```

`(run)`を評価してJettyを起動します。

``` bash
cemerick.austin.bcrepl-sample=> (run)
2014-09-24 14:10:43.857:INFO:oejs.Server:jetty-7.6.8.v20121106
2014-09-24 14:10:43.886:INFO:oejs.AbstractConnector:Started SelectChannelConnector@0.0.0.0:8080
#<Server org.eclipse.jetty.server.Server@2f07e79b>
cemerick.austin.bcrepl-sample=>
```

AustinのBrowser-REPL環境を作成します。Dockerコンテナの外からアクセスできるように`repl-env`の引数にIPアドレスを指定します。

``` bash
cemerick.austin.bcrepl-sample=> (def repl-env (reset! cemerick.austin.repls/browser-repl-env (cemerick.austin/repl-env :host "172.17.1.128")))
Browser-REPL ready @ http://172.17.1.128:43388/8371/repl/start
#'cemerick.austin.bcrepl-sample/repl-env
cemerick.austin.bcrepl-sample=>
```

作成したREPL環境を使い、ClojureScriptのREPLを起動します。

``` bash
cemerick.austin.bcrepl-sample=> (cemerick.austin.repls/cljs-repl repl-env)
Type `:cljs/quit` to stop the ClojureScript REPL
nil
cljs.user=>
```

ブラウザを操作するClojureScriptを実行します。

``` bash
cljs.user=> (js/alert "Salut!")
nil
```

ブラウザの画面にalertが表示されました。

{% img center /2014/09/25/clojurescript-repl-examples/austin.png %}



