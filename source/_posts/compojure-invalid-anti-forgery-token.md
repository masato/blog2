title: "CompojureのCSRF対策トークンのanti-forgery-token作成"
date: 2015-05-17 15:34:02
tags:
 - Compojure
 - Clojure
 - DockerCompose
 - CSRF
description: 前回Docker Composureを使って作成したCompojureプロジェクトに、curlからPOSTするとInvalid anti-forgery tokenが発生してしまいました。デフォルトでCSRF対策用トークンを追加しないとPOSTはできない仕様になっています。開発中はanti-forgeryを無効にした方が簡単です。いろいろ調べるとStackOverflowにあった方法でうまくいきました。
---

[前回](/2015/05/16/docker-compose-clojure/)Docker Composureを使って作成したCompojureプロジェクトに、curlからPOSTすると`Invalid anti-forgery token`が発生してしまいました。デフォルトでCSRF対策用トークンを追加しないとPOSTはできない仕様になっています。開発中は`anti-forgery`を無効にした方が簡単です。いろいろ調べるとStackOverflowにあった方法でうまくいきました。


<!-- more -->

## Invalid anti-forgery token

以下のようにPOST用ハンドラをhandler.cljに作成してcurlでPOSTすると`Invalid anti-forgery token`が発生します。

``` bash
$ curl -X POST -d "name=masato" localhost:3000/greeting
<h1>Invalid anti-forgery token</h1>
```

StackOverflowでも議論されていました。以下のサイトを参考に作業していきます。

* [How can I use ring anti-forgery / CSRF token with latest version ring/compojure?](http://stackoverflow.com/questions/30172569/how-can-i-use-ring-anti-forgery-csrf-token-with-latest-version-ring-compojure)

* [Set Ring-Anti-Forgery CSRF header token](http://stackoverflow.com/questions/20430281/set-ring-anti-forgery-csrf-header-token)

* [edbond/CSRF](https://github.com/edbond/CSRF)
 

## プロジェクト

前回と同じようにDocker Composeでプロジェクトを管理します。

``` bash
$ cd ~/clojure_apps
$ tree .
.
├── Dockerfile
├── cookies
├── docker-compose.yml
└── m2
```

Dockerホストの作業ユーザーと同じuidのユーザーをDockerコンテナにも作成します。MavenのjarファイルはVOLUMEに指定してDockerホストにマップしてコンテナ間で共有します。

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

``` bash
$ docker build -t clojure .
```

docker-compose.ymlの`working_dir`ディレクティブはCompojureプロジェクトを作成する前はコメントアウトしておきます。

```yaml ~/clojure_apps/docker-compose.yml
lein: &defaults
  image: clojure
  volumes:
    - .:/usr/src/app
    - ./m2:/home/docker/.m2
#  working_dir: /usr/src/app/hello-world
  ports:
    - "3000:3000"
    - "3449:3449"
cljslein:
  <<: *defaults
  ports:
    - "9000:9000"
```

`lein new`でcompojureのプロジェクトを作成します。

``` bash
$ cd ~/clojure_apps
$ docker-compose run --rm --service-ports lein new compojure hello-world
```

作成したCompojureプロジェクトを`working_dir`ディレクティブに指定します。

```yaml ~/clojure_apps/docker-compose.yml
lein: &defaults
  image: clojure
  volumes:
    - .:/usr/src/app
    - ./m2:/home/docker/.m2
  working_dir: /usr/src/app/hello-world
  ports:
    - "3000:3000"
    - "3449:3449"
cljslein:
  <<: *defaults
  ports:
    - "9000:9000"
```

project.cljファイルにはJSON作成の[cheshire](https://github.com/dakrone/cheshire)を依存関係に追加します。

```clj ~/clojure_apps/hello-world/project.clj
(defproject hello-world "0.1.0-SNAPSHOT"
  :description "FIXME: write description"
  :url "http://example.com/FIXME"
  :min-lein-version "2.0.0"
  :dependencies [[org.clojure/clojure "1.6.0"]
                 [compojure "1.3.1"]
                 [ring/ring-defaults "0.1.2"]
                 [cheshire "5.4.0"]]
  :plugins [[lein-ring "0.8.13"]]
  :ring {:handler hello-world.handler/app}
  :profiles
  {:dev {:dependencies [[javax.servlet/servlet-api "2.5"]
                        [ring-mock "0.1.5"]]}})
```

GET "/"ハンドラでCSRFトークンを出力するように変更します。トークンは[Ring-Anti-Forgery](https://github.com/ring-clojure/ring-anti-forgery/blob/master/src/ring/util/anti_forgery.clj)の`*anti-forgery-token*`を評価します。動的なvarのスペシャル変数です。セッションに保存しているcookie-storeのキーはランダムで適当に作成しました。

またJSONでレスポンスを返すために[cheshire](https://github.com/dakrone/cheshire)のgenerate-stringを使います。

```clj ~/clojure_apps/hello-world/src/hello_world/handler.clj
(ns hello-world.handler
  (:require [compojure.core :refer :all]
            [compojure.route :as route]
            [ring.middleware.defaults :refer [wrap-defaults site-defaults]]
            [ring.middleware.session :refer [wrap-session]]
            [ring.middleware.anti-forgery :refer :all]
            [ring.middleware.session.cookie :refer (cookie-store)]
            [cheshire.core :refer :all]))

(defn greeting-handler [request]
  (let [name (get-in request [:params :name])]
    (str "Hi, " name)))

(defroutes app-routes
  (GET "/" [] (generate-string {:csrf-token
                                *anti-forgery-token*}))
  (POST "/greeting" [] greeting-handler)
  (route/not-found "Not Found"))

(def app
  (-> app-routes
   (wrap-defaults site-defaults)
   (wrap-session {:cookie-attrs {:max-age 3600}
                  :store (cookie-store {:key "ahY9poQuaghahc7I"})})))
```

Ringサーバーをheadlessで起動します。

``` bash
$ cd ~/clojure_apps
$ docker-compose run --rm --service-ports lein ring server-headless
...
SelectChannelConnector@0.0.0.0:3000
Started server on port 3000
```

最初に用意したハンドラを実行してCSRFトークンを出力します。

``` bash
$ curl -X GET --cookie-jar cookies "http://localhost:3000/"
{"csrf-token":"FxyHMRjb5I9wwxskEo2h2uXhtU/CNUo38xLDTa/2fJp7QhZ/Wo7hi4zRbey9yUZgRKfe3y1uS66K8+kA"}
```

JSONで取得したcsfr-tokenをHTTPヘッダの`X-CSRF-Token`に指定してcurlからPOSTすると成功します。

``` bash
$ curl -X POST  \
  --cookie cookies  \
  -F "name=masato"  \
  -H "X-CSRF-Token: FxyHMRjb5I9wwxskEo2h2uXhtU/CNUo38xLDTa/2fJp7QhZ/Wo7hi4zRbey9yUZgRKfe3y1uS66K8+kA" \
  "http://localhost:3000/greeting"
Hi, masato
```

## anti-forgeryを無効にする場合

CSRF対策を無効にする場合は、`site-defaults`のanti-forgeryキーの値をfalseにして起動します。

```clj ~/clojure_apps/hello-world/src/hello_world/handler.clj
...
(def app (wrap-defaults app-routes (assoc-in site-defaults [:security :anti-forgery] false)))
```
