title: "ReagentでClojureScript REPLをリモートから操作する"
date: 2015-05-19 14:31:52
tags:
 - ClojureScript
 - Clojure
 - Reagent
 - React
 - REPL
description: ReagentはFacebook ReactのClojureScriptで書かれたインタフェースです。OmよりもよりClojureらしく書けるので使いやすいです。Reagentを使うとClojureScriptのREPLが使えるのですが、devモードではブラウザはlocalhostとWebSocketで通信するようになっています。開発はクラウド上で行っているのでdevモードでもリモートから操作できる必要があるので動作する環境を作ってみました。
---

[Reagent](http://holmsand.github.io/reagent/)はFacebook [React](https://facebook.github.io/react/)のClojureScriptで書かれたインタフェースです。[Om](https://github.com/swannodette/om)よりもよりClojureらしく書けるので使いやすいです。Reagentを使うとClojureScriptのREPLが使えるのですが、devモードではブラウザはlocalhostとWebSocketで通信するようになっています。開発はクラウド上で行っているのでdevモードでもリモートから操作できる必要があるので動作する環境を作ってみました。

<!-- more -->

## プロジェクト作成

いつものようにDocker Composeを使って開発環境を作ります。

``` bash
$ cd ~/clojure_apps
$ tree .
.
├── Dockerfile
├── cookies
├── docker-compose.yml
└── m2
```

今回はデバッグができるようにDockerfileを修正してnetstatをインストールしておきました。

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

Dockerイメージをビルドしておきます。

```bash
$ cd ~/clojure_apps
$ docker build -t clojure .
```

### Leiningenのreagent-template

Leiningenで作成するプロジェクトのテンプレートは[reagent-template](https://github.com/reagent-project/reagent-template)を使います。docker-component.ymlに`figwheel`サービスと`lein`サービスを作成します。

```yaml ~/clojure_apps/docker-component.yml
figwheel: &defaults
  image: clojure
  volumes:
    - .:/usr/src/app
    - ./m2:/home/docker/.m2
#  working_dir: /usr/src/app/hello-reagent
  ports:
    - "3449:3449"
lein:
  <<: *defaults
  ports:
    - "3000:3000"
```

`working_dir`ディレクティブをコメントアウトしてから`lein new`でプロジェクトを作成します。

``` bash
$ cd ~/cloure_apps
$ docker-compose run --rm --service-ports lein new reagent hello-reagent
Retrieving reagent/lein-template/0.8.1/lein-template-0.8.1.pom from clojars
Retrieving reagent/lein-template/0.8.1/lein-template-0.8.1.jar from clojars
Generating fresh 'lein new' Reagent project.
Removing clojureapps_lein_run_1...
```

## コードの修正

[reagent-template](https://github.com/reagent-project/reagent-template)の[dev.cljs](https://github.com/reagent-project/reagent-template/blob/master/src/leiningen/new/reagent/env/dev/cljs/reagent/dev.cljs)はlocalhostがハードコードされているのでproject.cljにwebsocket-urlやwebsocket-hostを指定しても反映されません。作成された`dev.cljs`を手動でpublicなIPアドレスに変更します。

```clj ~/clojure_apps/hello-reagent/env/dev/cljs/hello_reagent/dev.cljs
(ns ^:figwheel-no-load hello-reagent.dev
  (:require [hello-reagent.core :as core]
            [figwheel.client :as figwheel :include-macros true]
            [weasel.repl :as weasel]
            [reagent.core :as r]))

(enable-console-print!)

(figwheel/watch-and-reload
  :websocket-url "ws://210.xxx.xxx.xxx:3449/figwheel-ws"
  :jsload-callback core/mount-root)

;;(weasel/connect "ws://localhost:9001" :verbose true)

(core/init!)
```

[Figwheel](https://github.com/bhauman/lein-figwheel)はClojureScriptを自動でリロードしてくれる便利なツールです。Emacsでファイルを編集するとブラウザでClojureマークのアイコンが表示されてリロードされるので楽しくなります。最近Figwheelが3449ポートでREPLも提供してくれるみたいなので[Weasel](https://github.com/tomjakubowski/weasel)のconnectはコメントアウトしました。

project.cljのlein-figwheelのバージョンを`0.3.3`にあげます。


```clj ~/clojure_apps/hello-reagent/project.clj

                   :plugins [[lein-figwheel "0.3.3"]
                             [lein-cljsbuild "1.0.5"]]
```

## Figwheelの起動

docker-compose.ymlの`working_dir`ディレクティブにleinコマンドで作成したReagentプロジェクトを指定します。

```yaml ~/clojure_apps/docker-component.yml
figwheel: &defaults
  image: clojure
  volumes:
    - .:/usr/src/app
    - ./m2:/home/docker/.m2
  working_dir: /usr/src/app/hello-reagent
  ports:
    - "3449:3449"
lein:
  <<: *defaults
  ports:
    - "3000:3000"
```

Figwheelサーバーを起動します。

``` bash
$ cd ~/clojure_apps
$ docker-compose run --rm --service-ports lein figwheel
...
Started Figwheel autobuilder

Launching ClojureScript REPL for build: app
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

この後ブラウザから接続するとClojure ScriptのREPLが起動します。

```bash
To quit, type: :cljs/quit
cljs.user=>
```

Chromeの開発者ツールを開くとWebSocketの接続を確認できます。

http://210.xxx.xxx.xxx:3449/

![reagent-chrome.png](/2015/05/19/reagent-figwheel-clojurescript-repl/reagent-chrome.png)


REPLにJavaScriptのアラートを表示してみます。

``` bash
cljs.user=> (js/alert "REPLからアラート表示")
```

iTermで入力した値がChromeブラウザのアラートに表示されました。

![repl-alert.png](/2015/05/19/reagent-figwheel-clojurescript-repl/repl-alert.png)

