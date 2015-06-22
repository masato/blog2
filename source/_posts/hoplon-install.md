title: "Hoplon入門- Part1: Bootのバージョン"
date: 2015-06-15 10:04:59
tags:
 - Hoplon
 - Clojure
 - ClojureScript
 - Boot
 - DockerCompose
description: HoplonはClojureとClojureScriptで書けるフルスタックなWebフレームワークです。ClojureのビルドツールにはBootを使います。Bootには1.x系と2.x系がありますが残念ながら後方互換性がありません。現状ではHoplonはBoot 1で動作します。バージョンの問題で少し嵌まりますがClojureはScalaほど消耗しません。
---

[Hoplon](http://hoplon.io/)はClojureとClojureScriptで書けるフルスタックなWebフレームワークです。ClojureのビルドツールにはBootを使います。Bootには[1.x系](https://github.com/tailrecursion/boot/)と[2.x系](https://github.com/boot-clj/boot)がありますが残念ながら後方互換性がありません。現状ではHoplonはBoot 1で動作します。バージョンの問題で少し嵌まりますがClojureはScalaほど消耗しません。

<!-- more -->

## プロジェクト

適当なディレクトリにHoplon用のプロジェクトを作成します。

```bash
$ cd ~/clojure_apps/
$ tree -L 1
.
├── Dockerfile
├── docker-compose.yml
└── m2
```

### Dockerfile

ビルドツールはLeiningenとBootの1.x系と2.x系をインストールします。Bootのコマンドはそれぞれ`boot1`と`boot2`としています。デフォルトではBootは非rootユーザーが推奨されるためDockerには作業ユーザーの`docker`を作成しています。


```bash ~/clojure_apps/Dockerfile
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
```

### Hoplonで使うBootについて

Hoplonは現状Boot 1で動作します。[Runtime Exception #41](https://github.com/tailrecursion/boot/issues/41)にissueがあります。残念ながらBootには後方互換性がありません。Boot2を使うと`java.lang.RuntimeException`が発生します。

### docker-compose.yml

Docker Composeの設定ファイルです。boot1サービスではHoplonのデフォルト8000ポートをpotsディレクティブに指定します。

```~/clojure_apps/docker-compose.yml
lein: &defaults
  image: clojure
  volumes:
    - .:/usr/src/app
    - ./m2:/home/docker/.m2
boot1:
  <<: *defaults
  entrypoint: ["boot1"]
  ports:
    - "8000:8000"
```

## アプリの作成

HoplonアプリのビルドツールはBootですがテンプレートはLeiningenを使います。アプリ名は`spike-hoplon`としました。

``` bash
$ lein new hoplon spike-hoplon
```

Docker Composeを使う場合は以下を実行します。

```bash
$ docker-compose run --rm lein new hoplon spike-hoplon
```

[Getting Started](http://hoplon.io/#/getting-started/)の説明とは構成が異なりますが、以下のディレクトリ構造ができました。

```bash
$ cd ~/clojure_apps/spike-hoplon/
$ tree 
.
├── README.md
├── build.boot
└── src
    ├── index.cljs.hl
    └── main.inc.css
```

作成したプロジェクトはdocker-compose.ymlのworking_dirに指定します。

```~/clojure_apps/docker-compose.yml
lein: &defaults
  image: clojure
  volumes:
    - .:/usr/src/app
    - ./m2:/home/docker/.m2
  working_dir: /usr/src/app/spike-hoplon
boot1:
  <<: *defaults
  entrypoint: ["boot1"]
  ports:
    - "8000:8000"
```

## アプリの起動

build.bootファイルではhoplonのバージョンが`6.0.0-alpha2`になっています。このまま起動すると`java.io.FileNotFoundException`が発生します。[The getting started example doesn't seem to work. #64](https://github.com/tailrecursion/hoplon/issues/64)にあるissueのようにバージョンを`5.10.25`に下げます。


```clj ~/clojure_apps/spike-hoplon/build.boot
...
  :dependencies '[[tailrecursion/boot.task   "2.2.4"]
                  [tailrecursion/hoplon      "5.10.25"]]
```

`5.10.25`の[project.clj](https://github.com/tailrecursion/hoplon/blob/5.10.25/project.clj)は以下のようになっています。

```clj
(defproject tailrecursion/hoplon "5.10.25"
  :description  "Hoplon web development environment."
  :url          "http://github.com/tailrecursion/hoplon"
  :license      {:name "Eclipse Public License"
                 :url "http://www.eclipse.org/legal/epl-v10.html"}
  :plugins      [[lein-marginalia            "0.7.1"]]
  :dependencies [[io.hoplon.vendor/jquery    "1.8.2-0"]
                 [org.clojure/tools.reader   "0.8.5"]
                 [tailrecursion/javelin      "3.6.3"]
                 [tailrecursion/castra       "2.2.2"]
                 [clj-tagsoup                "0.3.0"]
                 [org.clojure/core.incubator "0.1.2"]
                 [org.clojure/clojurescript  "0.0-2234"]])
```

leinの[hoplon-template](https://github.com/tailrecursion/hoplon-template)の[hoplon.clj](https://github.com/tailrecursion/hoplon-template/blob/master/src/leiningen/new/hoplon.clj)を読むとbuild.bootファイルの生成時に`tailrecursion/hoplon`のバージョンを設定しています。バージョンは[ancient-clj](https://github.com/xsc/ancient-clj)を使い、Mavenリポジトリから[最新のバージョン](https://clojars.org/tailrecursion/hoplon)を取得しているようです。

```clj hoplon.clj
(def deps
  '[tailrecursion/boot.core
    tailrecursion/boot.task
    tailrecursion/hoplon])

(defn latest-deps-strs [deps]
  (mapv #(latest-version-string! % {:snapshots? false}) deps))
```

Hoplonは`6.0.0-alpha1`からBoot 2になり新しい書式のbuild.bootにあわせディレクトリ構造も変わっています。[hoplon-template](https://github.com/tailrecursion/hoplon-template)が生成するbuild.bootはBoot 1の書式なのでビルドに失敗しています。leinのhoplon-templateを使わない方が良さそうですが、初めてなのでこのままBootのdevelopmentタスクを実行してJettyを起動してみます。

```bash
$ boot1 development
```

Docker Composeのrunコマンド使う場合は`--service-ports`フラグを付けて起動します。

```bash
$ docker-compose run --rm --service-ports boot1 development
Compiling Hoplon dependencies...
Jetty server stored in atom here: #'tailrecursion.boot.task.ring/server...
Compiling Hoplon pages...
• src/index.cljs.hl
Compiling ClojureScript...
2015-06-15 01:44:32.185:INFO:oejs.Server:jetty-7.6.8.v20121106
2015-06-15 01:44:32.258:INFO:oejs.AbstractConnector:Started SelectChannelConnector@0.0.0.0:8000
↳ Elapsed time: 27.392 sec › 00:01:19
```

ブラウザーから起動を確認します。

![hoplon-hello.png](/2015/06/15/hoplon-install/hoplon-hello.png)
