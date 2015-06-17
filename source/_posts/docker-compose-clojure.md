title: "Docker Composeを使ってClojure開発環境を作る"
date: 2015-05-16 17:16:17
tags:
 - Docker
 - DockerCompose
 - Clojure
 - ClojureScript
 - Compojure
description: 前回Docker ComposeでSailsの開発環境を用意しました。Clojureの勉強用環境もDocker Composeで作ってみます。開発用のJavaはDockerコンテナを使いながら、ホストマシンのEmacsを使ってコードを編集できるようにします。

---

前回Docker Composeで[Sailsの開発環境](/2015/04/30/sails-pm2-docker-compose/)を用意しました。Clojureの勉強用環境もDocker Composeで作ってみます。開発用のJavaはDockerコンテナを使いながら、ホストマシンのEmacsを使ってコードを編集できるようにします。

<!-- more -->

## プロジェクト

プロジェクトのディレクトリを作成します。Dockerfile、docker-compose.yml、m2ディレクトリを作成します。

``` bash
$ cd ~/clojure_apps
$ tree .
.
├── Dockerfile
├── docker-compose.yml
└── m2
```

DockerとDocker Composeのバージョンを確認しておきます。

``` bash
$ docker version
...
Server version: 1.6.0
$ docker-compose --version
docker-compose 1.2.0
```

### Dockerfile

ベースイメージはオフィシャルの[clojure](https://registry.hub.docker.com/_/clojure/)を使います。Dockerホストの作業ユーザーと同じuidでユーザーを作成して、ボリューム内に作成されたファイルを編集できるようにします。

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

Dockerイメージは開発中も変更がないのでローカルにビルドしておきます。

``` bash
$ cd ~/clojure_apps
$ docker build -t clojure .
```

## REPL

Clojure REPLを実行するための基本的なdocker-compose.ymlを用意します。`WORKDIR`の`/usr/src/app`をカレントディレクトリにマップします。

``` yaml ~/clojure_apps/docker-compose.yml
lein: &defaults
  image: clojure
  volumes:
    - .:/usr/src/app
    - ./m2:/home/docker/.m2
```

Docker Composeのleinサービスをワンショットで実行します。初回実行時に`/home/docker/.m2`にMavenよりjarファイルがダウンロードされます。`.m2`ディレクトリはDockerホストにマップしているので2回目以降からはダウンロード済みのjarファイルを使います。

``` bash
$ cd ~/clojure_apps
$ docker-compose run --rm lein repl
...
nREPL server started on port 54118 on host 127.0.0.1 - nrepl://127.0.0.1:54118
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

## Compojure

Webフレームワークの[Compojure](https://github.com/weavejester/compojure)を使ったプロジェクトを作成してみます。

``` bash
$ cd ~/clojure_apps
$ docker-compose run --rm lein new compojure hello-world
Retrieving compojure/lein-template/0.4.2/lein-template-0.4.2.pom from clojars
Retrieving compojure/lein-template/0.4.2/lein-template-0.4.2.jar from clojars
Removing clojureapps_lein_run_1...
```

カレントディレクトリにCompojureのプロジェクトが作成されました。

``` bash
$ tree hello-world/
hello-world/
├── README.md
├── project.clj
├── resources
│   └── public
├── src
│   └── hello_world
│       └── handler.clj
└── test
    └── hello_world
        └── handler_test.clj
```

Docker ComposeからCompojureを起動するためにdocker-compose.ymlを編集します。`working_dir`にComposeプロジェクトのディレクトリを指定します。

``` yaml ~/clojure_apps/docker-compose.yml
lein: &defaults
  image: clojure
  volumes:
    - .:/usr/src/app
    - ./m2:/home/docker/.m2
  working_dir: /usr/src/app/hello-world
  ports:
    - "3000:3000"
```

[Ring](https://github.com/ring-clojure/ring)サーバーを起動します。`lein ring server`で起動すると`No X11 DISPLAY variable was set`とエラーがでてしまいます。ヘッドレスなDockerなので`lein ring server-headless`で起動します。また`--service-ports`フラグを指定してDockerホストにポートをマップします。

``` bash
$ docker-compose run --rm --service-ports lein ring server-headless
...
2015-05-16 09:42:00.573:INFO:oejs.Server:jetty-7.6.8.v20121106
2015-05-16 09:42:00.608:INFO:oejs.AbstractConnector:Started SelectChannelConnector@0.0.0.0:3000
Started server on port 3000
```

初回起動時にはjarファイルをダウンロードするためすこし時間がかかります。Dockerホストで別のシェルを起動してサーバーの起動を確認します。

```bash
$ curl -s localhost:3000
Hello World
```

## Clojure Script REPL

CompojureプロジェクトでClojure Script REPLを使ってみます。`project.clj`に`:plugin`と`:cljsbuild`ディレクティブを追加します。Clojureは1.6を使っているのでClojure Scriptは1.6系の`0.0-3126`を指定します。

```clj ~/clojure_apps/hello-world/project.clj
(defproject hello-world "0.1.0-SNAPSHOT"
  :description "FIXME: write description"
  :url "http://example.com/FIXME"
  :min-lein-version "2.0.0"
  :dependencies [[org.clojure/clojure "1.6.0"]
                 [compojure "1.3.1"]
                 [ring/ring-defaults "0.1.2"]
                 [org.clojure/clojurescript "0.0-3126"]]
  :plugins [[lein-ring "0.8.13"]
            [lein-cljsbuild "1.0.6"]]
  :ring {:handler hello-world.handler/app}
  :profiles
  {:dev {:dependencies [[javax.servlet/servlet-api "2.5"]
                        [ring-mock "0.1.5"]]}}
  :cljsbuild {
              :builds [{
                        :source-paths ["src-cljs"]
                        :compiler {
                                   :output-to "resources/public/main.js"
                                   :optimizations :whitespace
                                   :pretty-print true}}]}))
```

docker-compose.ymlはClojure Script用のサービスを`cljslein`として`lein`サービスをマージして用意します。

``` yaml ~/clojure_apps/docker-compose.yml
lein: &defaults
  image: clojure
  volumes:
    - .:/usr/src/app
    - ./m2:/home/docker/.m2
  working_dir: /usr/src/app/hello-world
  ports:
    - "3000:3000"
cljslein:
  <<: *defaults
  ports:
    - "9000:9000"
```

Docker Composeからcljsleinサービスを実行します。

``` bash
$ docker-compose run --rm --service-ports cljslein trampoline cljsbuild repl-rhino
Running Rhino-based ClojureScript REPL.
...
Retrieving org/hamcrest/hamcrest-core/1.1/hamcrest-core-1.1.jar from central
Retrieving org/clojure/tools.reader/0.8.16/tools.reader-0.8.16.jar from central
To quit, type: :cljs/quit
ClojureScript:cljs.user>
```

Clojure Script REPLからJavaScriptのDate関数を呼んでみます。

``` bash
ClojureScript:cljs.user> (js/Date)
#inst "2015-05-16T10:17:55.200-00:00"
```