title: "Figwheelで最新のClojureScriptの開発方法を勉強をする"
date: 2015-05-23 19:50:14
tags:
 - Figwheel
 - ClojureScript
 - Sablono
 - React
description: ClojureScriptやそのエコシステムは絶賛開発中のためライブラリの更新がとても盛んに行われています。2015年に出版されている英語の書籍でもすでに情報が古くなっています。最新の情報をネットで調べてもバージョンが違うと内容が異なっているので自分で手を動かして確認する必要があります。FigwheelのQuick Startを例にして最新のClojureScriptの開発方法を勉強していきます。
---

ClojureScriptやそのエコシステムは絶賛開発中のためライブラリの更新がとても盛んに行われています。2015年に出版されている英語の書籍でもすでに情報が古くなっています。最新の情報をネットで調べてもバージョンが違うと内容が異なっているので自分で手を動かして確認する必要があります。[Figwheel](https://github.com/bhauman/lein-figwheel)の[Quick Start](https://github.com/bhauman/lein-figwheel/wiki/Quick-Start)を例にして最新のClojureScriptの開発方法を勉強していきます。

<!-- more -->

## Figwheel

[Figwheel](https://github.com/bhauman/lein-figwheel)はClojureScritのコードをローカルで編集するとブラウザに自動的にプッシュして最新の状態にしてくれる[Leiningen](http://leiningen.org/)のプラグインです。最近はREPL機能も持つようになってさらに便利になっています。

### figwheel templateでプロジェクトの作成

`lein new`コマンドでfigwheel templateを使いプロジェクトを作成します。

``` bash
$ lein new figwheel hello-cljs
```

templateで作成されるproject.cljのデフォルトは以下のようになっています。[Quick Start](https://github.com/bhauman/lein-figwheel/wiki/Quick-Start)も現在最新版に修正中のようですが内容が古くなっています。

```clj ~/clojure_apps/hello-cljs/project.clj
(defproject hello-cljs "0.1.0-SNAPSHOT"
  :description "FIXME: write this!"
  :url "http://example.com/FIXME"
  :license {:name "Eclipse Public License"
            :url "http://www.eclipse.org/legal/epl-v10.html"}

  :dependencies [[org.clojure/clojure "1.6.0"]
                 [org.clojure/clojurescript "0.0-3211"]
                 [org.clojure/core.async "0.1.346.0-17112a-alpha"]]

  :plugins [[lein-cljsbuild "1.0.5"]
            [lein-figwheel "0.3.1"]]

  :source-paths ["src"]

  :clean-targets ^{:protect false} ["resources/public/js/compiled" "target"]

  :cljsbuild {
    :builds [{:id "dev"
              :source-paths ["src"]
              :figwheel { :on-jsload "hello-cljs.core/on-js-reload" }

              :compiler {:main hello-cljs.core
                         :asset-path "js/compiled/out"
                         :output-to "resources/public/js/compiled/hello_cljs.js"
                         :output-dir "resources/public/js/compiled/out"
                         :source-map-timestamp true }}
             {:id "min"
              :source-paths ["src"]
              :compiler {:output-to "resources/public/js/compiled/hello_cljs.js"
                         :main hello-cljs.core
                         :optimizations :advanced
                         :pretty-print false}}]}
  :figwheel {
             ;; :http-server-root "public" ;; default and assumes "resources"
             ;; :server-port 3449 ;; default
             :css-dirs ["resources/public/css"] ;; watch and update CSS
             })
```

以下のようにライブラリのバージョンを上げます。特に[Failed When Compiling by Using Cljs 0.0-3255 #393](https://github.com/emezeske/lein-cljsbuild/issues/393)にあるissueのように、ClojureScriptｍの`0.0-3255`以降ではClojureのバージョンは`1.7.0-beta2`以降が必要です。

`:cljsbuild`ディレクティブではクラウド上の開発環境にローカルのブラウザから接続できるようにWebSocketのホストをパブリックIPアドレスに指定しています。

```clj ~/clojure_apps/hello-cljs/project.clj
...
  :dependencies [[org.clojure/clojure "1.7.0-beta3"]
                 [org.clojure/clojurescript "0.0-3269"]
                 [org.clojure/core.async "0.1.346.0-17112a-alpha"]]

  :plugins [[lein-cljsbuild "1.0.5"]
            [lein-figwheel "0.3.3"]]
...
  :cljsbuild {
    :builds [{:id "dev"
              :source-paths ["src"]

              :figwheel { :on-jsload "hello-cljs.core/on-js-reload"
                          :websocket-host "210.xxx.xxx.xxx"}
```

### FigwheelのWebサーバーを起動

`lein figwheel`コマンドでのWebサーバーを起動します。

```bash
$ lein figwheel
Figwheel: Starting server at http://localhost:3449
Focusing on build ids: dev
Compiling "resources/public/js/compiled/hello_cljs.js" from ["src"]...
Successfully compiled "resources/public/js/compiled/hello_cljs.js" in 14.0 seconds.
Started Figwheel autobuilder

Launching ClojureScript REPL for build: dev
Figwheel Controls:
          (stop-autobuild)                ;; stops Figwheel autobuilder
          (start-autobuild [id ...])      ;; starts autobuilder focused on optional ids
          (switch-to-build id ...)        ;; switches autobuilder to different build
          (reset-autobuild)               ;; stops, cleans, and starts autobuilder
          (build-once [id ...])           ;; builds source one time
          (clean-builds [id ..])          ;; deletes compiled cljs target files
          (fig-status)                    ;; displays current state of system
          (add-dep [org.om/om "0.8.1"]) ;; add a dependency. very experimental
  Switch REPL build focus:
          :cljs/quit                      ;; allows you to switch REPL to another build
    Docs: (doc function-name-here)
    Exit: Control+C or :cljs/quit
 Results: Stored in vars *1, *2, *3, *e holds last exception object
Prompt will show when figwheel connects to your application
```

ローカルのブラウザでFigwheelのサーバーに確認します。

http://210.xxx.xxx.xxx:3449/


### REPL

ブラウザからWebSocketの接続ができるとREPLが使えるようになります。

``` bash
Prompt will show when figwheel connects to your application

To quit, type: :cljs/quit
cljs.user=> cljs.user=>
```

JavaScriptのDate関数をnewしてみます。

``` clj
cljs.user=> (js/Date.)
#inst "2015-05-24T13:20:18.481-00:00"
```

Linuxで動作しているREPLからJavaScriptのアラートをブラウザに表示することもできます。

``` clj
cljs.user=> (js/alert "browserにアラートがでます。")
```

ただしREPLはreadlineが有効になっていないため上アローキーや`C-p`でヒストリーバックが効きません。[rlwrap](https://github.com/hanslub42/rlwrap)でleinコマンドをラップすると良いらしいですが、Docker Composeで実行するとうまく動作できませんでした。

```bash
$ docker-compose run --rm --service-ports lein figwheel
rlwrap: error: My terminal reports width=0 (is it emacs?)  I can't handle this, sorry!
Removing clojureapps_lein_run_1...
```

## Quick Startと違うところ

figwheel templateで作成されるcore.cljsのデフォルトです。

```clj ~/clojure_apps/hello-cljs/src/hello_cljs/core.cljs
(ns ^:figwheel-always hello-cljs.core
    (:require))

(enable-console-print!)

(println "Edits to this text should show up in your developer console.")

;; define your app data so that it doesn't get over-written on reload

(defonce app-state (atom {:text "Hello world!"}))


(defn on-js-reload []
  ;; optionally touch your app-state to force rerendering depending on
  ;; your application
  ;; (swap! app-state update-in [:__figwheel_counter] inc)
)
```

### (enable-console-print!)

Quick StartではJavaScriptのconsoleを使う例になっています。

```clj
(.log js/console "Hey Seymore! wts goin' on?")
```

figwheel templateでは`(enable-console-print!)`が有効になっています。ClojureScriptの[core.cljs](https://github.com/clojure/clojurescript/blob/master/src/main/cljs/cljs/core.cljs#L110)で定義されている関数です。

```clsj
(defn enable-console-print!
  "Set *print-fn* to console.log"
  []
  (set! *print-newline* false)
  (set! *print-fn*
    (fn [& args]
      (.apply (.-log js/console) js/console (into-array args)))))
```

ちなみにスレッド毎に動的束縛が使えるな変数をアスタリスクで囲うことを`earmuff`と呼ぶそうです。かわいい。。

println関数を使ってブラウザのコンソールにログを出力してみます。

```clj ~/clojure_apps/hello-cljs/src/hello_cljs/core.cljs
(ns ^:figwheel-always hello-cljs.core
    (:require))

(enable-console-print!)
(println "enable-console-print!を実行するとprintlnが使えます")
)
```

![figwheel-println.png](/2015/05/23/clojurescript-figwheel-developing/figwheel-println.png)

### on-js-reload関数

on-js-reload関数をコメントアウトするとフックが呼ばれなくなります。

```clj
(defn on-js-reload []
  ;; optionally touch your app-state to force rerendering depending on
  ;; your application
  ;; (swap! app-state update-in [:__figwheel_counter] inc)
)
```

ブラウザのコンソールに毎回ログがでるので残しておいた方がよさそうです。

```
Figwheel: :on-jsload hook 'hello-cljs.core/on-js-reload' is missing
```

### index.html

figwheel templateでは`id="app"`とCSSの読み込みがすでにが定義されています。

```html ~/clojure_apps/hello-cljs/resources/public/index.html
<!DOCTYPE html>
<html>
  <head>
    <link href="css/style.css" rel="stylesheet" type="text/css">
  </head>
  <body>
    <div id="app">
      <h2>Figwheel template</h2>
      <p>Checkout your developer console.</p>
    </div>
    <script src="js/compiled/hello_cljs.js" type="text/javascript"></script>
  </body>
</html>
```


## Quick Startしてみる

[Quick Start](https://github.com/bhauman/lein-figwheel/wiki/Quick-Start)の手順にしたがってプログラムを修正していきます。

### Sablono

project.cljに[Sablono](https://github.com/r0man/sablono)を追加します。Sablonoは[Hiccup](https://github.com/weavejester/hiccup)風に記述して[React](https://facebook.github.io/react/)用のHTMLをレンダリングできるテンプレートです。[Om]()

```clj ~/clojure_apps/hello-cljs/project.clj
  :dependencies [[org.clojure/clojure "1.7.0-beta3"]
                 [org.clojure/clojurescript "0.0-3269"]
                 [org.clojure/core.async "0.1.346.0-17112a-alpha"]
                 [sablono "0.3.4"]]
```

`lein clean`をしてから依存関係をダウンロードします。

```bash
$ lein clean
$ lein figwheel
Retrieving sablono/sablono/0.3.4/sablono-0.3.4.pom from clojars
Retrieving sablono/sablono/0.3.4/sablono-0.3.4.jar from clojars
```

### defonce

Quick Startでは以下のように最初は`def`で定義してブラウザでリロードするとstateが消えてしまう例が書いてあります。`defonce`に変更して状stateを保持するようになっています。

```clj
(def app-state (atom { :likes 0 }))
```

figwheel templateではデフォルトで`defonce`を使っています。

```clj ~/clojure_apps/hello-cljs/src/hello_cljs/core.cljs
(defonce app-state (atom {:text "Hello world!"}))
```

Quick Startのようにatomの変更して`:links`キーを持つマップにします。

```clj ~/clojure_apps/hello-cljs/src/hello_cljs/core.cljs
(defonce app-state (atom {:likes 0}))
```

`defonce`は[core.clj](https://github.com/clojure/clojurescript/blob/master/src/main/clojure/cljs/core.clj#L135)に定義されているマクロです。この辺りがClojureらしいところです。

```clj
(defmacro defonce [x init]
  `(when-not (exists? ~x)
     (def ~x ~init)))
```

core.cljsは以下のようになります。

```clj ~/clojure_apps/hello-cljs/src/hello_cljs/core.cljs
(ns ^:figwheel-always hello-cljs.core
    (:require [sablono.core :as sab]
              [hello-cljs.components :refer [like-seymore]]))

(enable-console-print!)
(println "enable-console-print!を実行するとprintlnが使えます")

(defonce app-state (atom {:likes 0}))

(defn like-seymore [data]
  (sab/html [:div
             [:h1 "いいね!の数: " (:links @data)]
             [:div [:a {:href "#"
                        :onClick #(swap! data update-in [:links] inc)}
                    "いいね!"]]]))

(defn render! []
  (.render js/React
           (like-seymore app-state)
           (.getElementById js/document "app")))
(add-watch app-state :on-change (fn [_ _ _ _ ] (render!)))

(render!)

(defn on-js-reload [])
```

ブラウザを開き、`いいね!`のリンクをクリックするとカウントアップします。cljsを編集してコードをプッシュしてもstateは保持されるので0に戻りません。

![5times.png](/2015/05/23/clojurescript-figwheel-developing/5times.png)

## 自動リロードの設定

### ^:figwheel-always

自動リロードの使い方もQuick Startとtemplateでは違いがあります。テンプレートではcore.cljsのnsの先頭に`^:figwheel-always`が入っています。

```clj ~/clojure_apps/hello-cljs/src/hello_cljs/core.cljs
(ns ^:figwheel-always hello-seymore.core
  (:require [sablono.core :as sab]
            [hello-seymore.components :refer [like-seymore]]))
```

`^:figwheel-always`をnsの先頭に記述するとcljsに変更があるとnamespaceのリロード対象になります。

### cljsの分割

like-saymore関数を切り出してcomponents.cljsを作成します。ns関数に`^:figwheel-always`を追加してリロードの対象に追加します。

```clj ~/clojure_apps/hello-cljs/src/hello_cljs/components.cljs
(ns ^:figwheel-always hello-cljs.components
    (:require [sablono.core :as sab]))

(defn like-seymore [data]
  (sab/html [:div
             [:h1 "いいね!の数: " (:links @data)]
             [:div [:a {:href "#"
                        :onClick #(swap! data update-in [:links] inc)}
                    "いいね!"]]]))
```



`^:figwheel-always`を使うとcomponents.cljsを編集しただけでもcore.jsもリロードされます。namespace全体がリロードされるのでプログラムが大きくなった時には遅くなりそうなので注意が必要です。

```
("hello_cljs/components.js" "hello_cljs/core.js")
```

### CSSのリロードとコンパイル済みのアセット

コンパイル済みのアセットを`resources/public`ディレクトリからサーブします。あわせてCSSを修正しても自動リロードされるようにします。こちらもテンプレートではデフォルトでproject.cljに設定が有効になっています。

* asset-path、outputo-to、output-dirの指定
* clean-targetsの指定

```clj ~/clojure_apps/hello-cljs/project.clj
...
  :clean-targets ^{:protect false} ["resources/public/js/compiled" "target"]

  :cljsbuild {
    :builds [{:id "dev"
              :source-paths ["src"]

              :figwheel { :on-jsload "test.core/on-js-reload" }

              :compiler {:main test.core
                         :asset-path "js/compiled/out"
                         :output-to "resources/public/js/compiled/test.js"
                         :output-dir "resources/public/js/compiled/out"
                         :source-map-timestamp true }}
  :figwheel {
             :css-dirs ["resources/public/css"] ;; watch and update CSS
  }
```

以下のようにCSSを記述すると自動的に変更がブラウザにプッシュされます。

```css clojure_apps/hello-cljs/resources/public/css/style.css
body {
    background-color: yellow;
}
```