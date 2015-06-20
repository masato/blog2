title: "Hoplon入門- Part2: Boot 2でビルドする"
date: 2015-06-16 12:34:14
tags:
 - Hoplon
 - Clojure
 - ClojureScript
 - Boot
description: 前回Hoplonのプロジェクトを作成するところまで動きました。ただしleinテンプレートのhoplon-templateが生成するコードはHoplon 5.10.25なので、index.cljs.hlの内容がGetting Startedと少し異なります。Hoplon 6とBoot 2の組み合わせを試してみようと思います。

---

[前回](/2015/06/15/hoplon-install/)Hoplonのプロジェクトを作成するところまで動きました。ただしleinテンプレートの[hoplon-template](https://github.com/tailrecursion/hoplon-template)が生成するコードはHoplon [5.10.25](https://github.com/tailrecursion/hoplon/tree/5.10.25)なので、[index.cljs.hl](https://github.com/tailrecursion/hoplon-template/blob/master/src/leiningen/new/hoplon/index.cljs.hl)の内容が[Getting Started](http://hoplon.io/#/getting-started/)と少し異なります。Hoplon 6とBoot 2の組み合わせを試してみようと思います。

<!-- more -->

## Boot 1のプロジェクト

Clojureの作業ディレクトリに移動してboot1コマンドから開発用のサーバーを起動します。

```bash
$ cd ~/clojure_apps/
$ docker pull clojure
$ docker build -t clojure .
$ docker-compose run --rm --service-ports boot1 development
```

[hoplon-template](https://github.com/tailrecursion/hoplon-template)が生成するHoplon 5.10.25 / Boot 1.1.1のディレクトリは以下のようになります。publicディレクトリにコンパイルされたファイルが生成されます。このディレクトリのファイルは手動で編集してもコンパイルの度に書き換えられます。

```bash
$ tree
.
├── README.md
├── build.boot
├── resources
│   └── public
│       ├── c6f4dce0-0384-11e4-9191-0800200c9a66.css
│       ├── c6f4dce0-0384-11e4-9191-0800200c9a66.js
│       └── index.html
└── src
    ├── index.cljs.hl
    └── main.inc.css
```


## Boot 2のプロジェクト

docker-compose.ymlにBoot 2の[boot.sh](https://github.com/boot-clj/boot/releases/download/2.0.0/boot.sh)を使うbootサービスを定義します。Boot 2を無印のデフォルトにしました。

```~/clojure_apps/docker-compose.yml
lein: &defaults
  image: clojure
  volumes:
    - .:/usr/src/app
    - ./m2:/home/docker/.m2
  working_dir: /usr/src/app/hoplon6-start
boot1:
  <<: *defaults
  entrypoint: ["boot1"]
  ports:
    - "8000:8000"
boot:
  <<: *defaults
  entrypoint: ["boot2"]
  ports:
    - 5000:5000
    - 4449:4449
bash:
  <<: *defaults
  entrypoint: ["bash"]
```

Clojure用のDockerイメージにはBoot 1の`/usr/local/bin/boot1`とBoot 2の`/usr/local/bin/boot2`コマンドをインストールしています。

```bash:~/clojure_apps/Dockerfile
FROM clojure
MAINTAINER Masato Shimizu <ma6ato@gmail.com>

WORKDIR /usr/src/app

ADD https://github.com/boot-clj/boot/releases/download/2.0.0/boot.sh /tmp/
RUN mv /tmp/boot.sh /usr/local/bin/boot2 && \
    chmod 755 /usr/local/bin/boot2

ADD https://clojars.org/repo/tailrecursion/boot/1.1.1/boot-1.1.1.jar /tmp/
RUN mv /tmp/boot-1.1.1.jar /usr/local/bin/boot1 && \
    chmod 755 /usr/local/bin/boot1

RUN adduser --disabled-password --gecos '' --uid 1000 docker && \
  mkdir /home/docker/.m2 && \
  chown -R docker:docker /usr/src/app /home/docker/.m2

VOLUME /home/docker/.m2
USER docker
RUN lein

ENTRYPOINT ["lein"]
CMD []
```

### bootコマンド

bootコマンドの使い方は[boot](https://github.com/boot-clj/boot)のREADMEに書いてあります。ヘルプを表示してみます。

```bash
$ cd ~/clojure_apps
$ docker-compose run --rm boot2 -h
Boot App Version: 2.0.0
Boot Lib Version: 2.1.2
Clojure Version:  1.6.0

Usage:   boot OPTS <task> TASK_OPTS <task> TASK_OPTS ...
```

REPLも使えます。

```bash
$ docker-compose run --rm boot repl
Retrieving tools.nrepl-0.2.8.jar from https://repo1.maven.org/maven2/
nREPL server started on port 37446 on host 127.0.0.1 - nrepl://127.0.0.1:37446
REPL-y 0.3.5, nREPL 0.2.8
Clojure 1.6.0
OpenJDK 64-Bit Server VM 1.7.0_79-b14        Exit: Control+D or (exit) or (quit)
    Commands: (user/help)
        Docs: (doc function-name-here)
              (find-doc "part-of-name-here")
Find by Name: (find-name "part-of-name-here")
      Source: (source function-name-here)
     Javadoc: (javadoc java-object-or-class-here)
    Examples from clojuredocs.org: [clojuredocs or cdoc]
              (user/clojuredocs name-here)
              (user/clojuredocs "ns-here" "name-here")
boot.user=>
```

### build.boot

最初に[hoplon-template](https://github.com/tailrecursion/hoplon-template)を使わないで、6.0.0-alpha1 / Boot 2.0.0に対応したプロジェクトを作成してみます。

```bash
$ cd ~/clojure_apps
$ mkdir hoplon6-start
$ cd !$
```

Bootの[README](https://github.com/boot-clj/boot)を参考にして簡単なプロジェクトを作成します。


```clj:~/clojure_apps/hoplon6-start/build.boot
(set-env!
  :source-paths #{"src"}
  :dependencies '[[me.raynes/conch "0.8.0"]])

(task-options!
  pom {:project 'my-project
       :version "0.1.0"}
  jar {:manifest {"Foo" "bar"}})
```

作成したbuild.bootを使い空のプロジェクトをビルドしてjarファイルを作成します。

```bash
$ docker-compose run --rm boot pom jar install
Retrieving conch-0.8.0.jar from https://clojars.org/repo/
Retrieving useful-0.10.6.jar from https://clojars.org/repo/
Retrieving tools.reader-0.7.2.jar from https://repo1.maven.org/maven2/
Writing pom.xml and pom.properties...
Writing my-project-0.1.0.jar...
Installing my-project-0.1.0.jar...
Removing clojureapps_boot2_run_1...
```

targetディレクトリにコンパイルされました。

```
$ tree hoplon6-start
hoplon6-start/
├── build.boot
└── target
    ├── META-INF
    │   └── maven
    │       └── my-project
    │           └── my-project
    │               ├── pom.properties
    │               └── pom.xml
    └── my-project-0.1.0.jar
```

シェルから実行した`boot pom jar install`のコマンドはREPLでもインタラクティブに実行できます。

```bash
$ docker-compose run --rm boot repl

boot.user=> (boot (pom) (jar) (install))
Writing pom.xml and pom.properties...
Writing my-project-0.1.0.jar...
Installing my-project-0.1.0.jar...
```

### タスクを作成する

`deftask`にbuildタスクを記述します。comp関数で複数のタスクで構成されるフローを1つのタスクにまとめることができます。composeするタスクのパイプラインは[Transducers](http://clojure.org/transducers)のように左から右に実行されるようです。

```clj:~/clojure_apps/hoplon6-start/build.boot
(set-env!
  :source-paths #{"src"}
  :dependencies '[[me.raynes/conch "0.8.0"]])

(task-options!
  pom {:project 'my-project
       :version "0.1.0"}
  jar {:manifest {"Foo" "bar"}})

(deftask build
  "Build my project."
  []
  (comp (pom) (jar) (install)))
```

## Boot 2 でHoplonをビルドする

Boot 2プロジェクトを以下のように作成しました。

```bash
$ tree -L 2 hoplon6-start/
hoplon6-start/
├── build.boot
└── src
    └── index.cljs.hl
```

index.cljs.hlはH1の見出しだけ書いてあります。

```clj:~/clojure_apps/hoplon6-start/src/index.cljs.hl
(page "index.html")

(html
  (head)
  (body
    (h1 "Hoplon Minimal!")))
```

コンパイルすると`target`ディレクトリにindex.htmlとindex.html.jsファイルが出力されます。

```bash
.
├── build.boot
├── build.boot.mini
├── src
│   └── index.cljs.hl
└── target
    ├── index.html
    ├── index.html.js
    └── out
```

### build.boot

Hoplonのbuild.bootをBoot 2で記述します。サンプルが少ないのですが、[build.boot](https://github.com/mynomoto/hoplon-minimal/blob/master/build.boot)を参考にして依存パッケージのバージョンをそれぞれ最新に変更しています。

```clj:~/clojure_apps/hoplon6-start/build.boot
(set-env!
  :dependencies  '[[org.clojure/clojure       "1.7.0-RC2"]
                   [adzerk/boot-cljs          "0.0-3308-0"]
                   [adzerk/boot-cljs-repl     "0.1.10-SNAPSHOT"]
                   [adzerk/boot-reload        "0.3.1"]
                   [pandeiro/boot-http        "0.6.3-SNAPSHOT"]
                   [tailrecursion/boot-hoplon "0.1.0-SNAPSHOT"]
                   [tailrecursion/hoplon      "6.0.0-alpha2"]]
  :source-paths   #{"src"})

(require
  '[adzerk.boot-cljs          :refer [cljs]]
  '[adzerk.boot-reload        :refer [reload]]
  '[pandeiro.boot-http        :refer [serve]]
  '[adzerk.boot-cljs-repl     :refer [cljs-repl start-repl]]
  '[tailrecursion.boot-hoplon :refer [hoplon prerender]])

(deftask dev
  "Build hoplon-minimal for local development."
  []
  (comp
    (serve :port 5000
           :httpkit true)
    (watch)
    (speak)
    (hoplon)
    ;(reload :port 4449)
    (cljs)))
```

### クライアントのライブリロード

[Figwheel](https://github.com/bhauman/lein-figwheel)の場合はproject.cljにライブリロードのためクライアントが接続するWebSocketのホストを指定できました。

```project.clj
  :cljsbuild {
    :builds [{:id "dev"
              :source-paths ["src"]

              :figwheel { :on-jsload "hello-cljs.core/on-js-reload"
                          :websocket-host "210.xxx.xxx.xxx"}
```

[adzerk-oss/boot-reload](https://github.com/adzerk-oss/boot-reload)の場合、`ip`オプションはWebSockertサーバーがBindするIPアドレスになります。publicのIPアドレスを指定できないため、デフォルトだと`ws://localhost:port`にブラウザから接続することになります。リモートからWebSocketサーバーに接続してライブリロードする設定がみつからないためとりあえずコメントアウトしました。


### boot dev

boooコマンドからdevタスクを実行します。サーバーの起動後にindex.cljs.hlを編集すると自動でコンパイルは行いますが、reloadはコメントアウトしているのでクライアント側のライブリロードはされません。

```
$ docker-compose run --rm --service-ports boot dev
temp-dir! was deprecated, please use tmp-dir! instead
<< started HTTP Kit on http://localhost:5000 >>

Starting file watcher (CTRL-C to quit)...

tmppath was deprecated, please use tmp-path instead
tmpfile was deprecated, please use tmp-file instead
Compiling Hoplon pages...
• index.cljs.hl
Compiling index.html.js...
Elapsed time: 25.526 sec

Compiling Hoplon pages...
• index.cljs.hl
Compiling index.html.js...
Elapsed time: 0.553 sec
```

まだ試行錯誤中ですがブラウザから画面表示を確認できました。

![hoplon-minimal.png](/2015/06/16/hoplon-install-boot2/hoplon-minimal.png)