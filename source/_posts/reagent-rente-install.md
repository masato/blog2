title: "Reagent入門 - Part5: Renteをインストール"
date: 2015-06-23 00:15:32
tags:
tags:
 - SPA
 - Clojure
 - ClojureScript
 - React
 - Rente
 - Sente
 - Figwheel
description: RenteはReagent(React)とSenteを使ったClojureScriptのフレームワークです。サーバーとの通信はWebSocketとAjax、core.asyncが使えます。デンマークのEnterlabという会社がcommitも盛んに開発しています。Eclipse Public License](https://ja.wikipedia.org/wiki/Eclipse_Public_License)です。
---

[Rente](https://github.com/enterlab/rente)は[Reagent(React)](https://github.com/reagent-project/reagent)と[Sente](https://github.com/ptaoussanis/sente)を使ったClojureScriptのフレームワークです。サーバーとの通信はWebSocketとAjax、[core.async](https://github.com/clojure/core.async)が使えます。デンマークの[Enterlab](http://enterlab.com/)という会社がcommitも盛んに開発しています。[Eclipse Public License](https://ja.wikipedia.org/wiki/Eclipse_Public_License)です。

<!-- more -->

## Clojureプロジェクトのおさらい

以前のDockerイメージは破棄して新しいプロジェクトを作ります。1ヶ月経つとClojure用のDockerfileとdocker-compose.ymlも少し変わりました。

### Dockerfileとdocker-compose.yml

ビルドツールは[Leiningen](http://leiningen.org/)に加えて[Boot](https://github.com/boot-clj/boot)もインストールして使えるようにしています。

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

ローカルのClojureイメージをビルドし直します。

```bash
$ cd ~/clojure_apps
$ docker pull clojure
$ docker build -t masato/clojure .
```

docker-compose.ymlでは、DockerコンテナのタイムゾーンをDockerホストに合わています。

```yaml:~/clojure_apps/docker-compose.yml
lein: &defaults
  image: masato/clojure
  volumes:
    - .:/usr/src/app
    - ./m2:/home/docker/.m2
    - /etc/localtime:/etc/localtime:ro
  working_dir: /usr/src/app/docker-rente
rente:
  <<: *defaults
  ports:
    - 8080:8080
figwheel:
  <<: *defaults
  ports:
    - 3449:3449
boot1:
  <<: *defaults
  entrypoint: ["boot1"]
  ports:
    - 8000:8000
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

`~/.bashrc`にdocker-composeコマンドのエイリアスを作成して再読込します。

```bash:~/.bashrc
alias lein='docker-compose run --rm --service-ports lein'
alias figwheel='docker-compose run --rm --service-ports figwheel'
alias rente='docker-compose run --rm --service-ports rente'
```

## Renteのインストール

### インストール

とりあえず[Rente](https://github.com/enterlab/rente)をインストールして動かしてみます。リポジトリをcloneします。

```bash
$ cd ~/clojure_apps
$ git clone https://github.com/enterlab/rente docker-rente
```

docker-compose.ymlの`working_dir`にcloneしたディレクトリを指定します。

```yaml:~/clojure_apps/docker-compose.yml
lein: &defaults
  image: masato/clojure
  volumes:
    - .:/usr/src/app
    - ./m2:/home/docker/.m2
    - /etc/localtime:/etc/localtime:ro
  working_dir: /usr/src/app/docker-rente
...
```

Dockerホストはクラウド上で起動しているので、リモートから[Figwheel](https://github.com/bhauman/lein-figwheel/)の3449ポートにWebSocketで接続できるようにpublic IPアドレスを指定します。

```clj:~/clojure_apps/docker-rente/dev/start.cljs
(ns rente.start
  (:require [figwheel.client :as fw]
            [rente.client.app :as app]))

(enable-console-print!)

(fw/watch-and-reload
 :websocket-url "ws://xxx.xxx.xxx.xxx:3449/figwheel-ws"
 :jsload-callback #(swap! app/state update-in [:re-render-flip] not))

(app/main)
```

### アプリの起動

ClojureScriptの開発用に[Figwheel](https://github.com/bhauman/lein-figwheel)を起動します。`~/.bashrc`に定義したdocker-compose用のエイリアスを使っているので、実際のコマンドは`lein figwheel`になります。ライブリロード用のWebSocketサーバーが3449ポートで起動します。

```bash
$ cd ~/clojure_apps
$ figwheel figwheel
...
Figwheel: Starting server at http://localhost:3449
Focusing on build ids: client
Compiling "resources/public/js/app.js" from ["src/rente/client" "dev"]...
Successfully compiled "resources/public/js/app.js" in 1.742 seconds.
Started Figwheel autobuilder
Figwheel: focusing on build-ids (client)
Compiling "resources/public/js/app.js" from ["src/rente/client" "dev"]...
Successfully compiled "resources/public/js/app.js" in 0.749 seconds.
notifying browser that file changed:  resources/public/js/app.js
notifying browser that file changed:  dev-resources/public/js/out/goog/deps.js
```

別のシェルを開いてアプリを実行します。こちらもdocker-composeのサービスなので実際には`lein run`を実行しています。

```bash
$ rente run
2015-06-22T14:34:25,607Z [main] INFO  rente.run - rente started
```

DockerホストにブラウザからRenteサーバーが起動している8080ポートに接続します。

http://xxx.xxx.xxx.xxx:8080/

![rente.png](/2015/06/23/reagent-rente-install/rente.png)

[Bootstrap](http://getbootstrap.com/)を使っているように見えませんが、3.3.4がロードされているようです。

### Send Message Callback

`Send Message Callback`ボタンを押して動作確認します。

テンプレートが生成したClojureScriptの`views.cljs`は以下のようになっています。ボタンの`on-click`イベントで`socket/test-socket-callback`が発火されています。

```clj:~/clojure_apps/docker-rente/src/rente/client/views.cljs
(ns rente.client.views
  (:require [rente.client.ws :as socket]))

(defn main [data]
  [:div
   [:h1 (:title @data)]
   [:span "Hello world! This is reagent!"]
   [:br]
   [:span "And sente seems to work too.."]
   [:br]
   [:span "And figwheel.. w00t!"]
   [:br]
   [:button {:on-click socket/test-socket-callback} "Send Message Callback"]
   [:br]
   [:button {:on-click socket/test-socket-event} "Send Message Event"]
   ])
```

コールバックを定義しているClojureScriptは以下です。

```clj:~/clojure_apps/docker-rente/src/rente/client/ws.cljs
(defn test-socket-callback []
  (chsk-send!
    [:rente/testevent {:message "Hello socket Callback!"}]
    2000
    #(js/console.log "CALLBACK from server: " (pr-str %))))
```

ボタンを押すとブラウザのコンソールにコールバックのメッセージが出力されます。

![rente-callback.png](/2015/06/23/reagent-rente-install/rente-callback.png)


### Send Message Event

`Send Message Event`も同様にコールバック関数は次のようになっています。


```clj:~/clojure_apps/docker-rente/src/rente/client/ws.cljs
(defn test-socket-event []
  (chsk-send! [:rente/testevent {:message "Hello socket Event!"}]))
```

ボタンを押すとブラウザのコンソールにコールバックのメッセージが出力されました。

![rente-pushevent.png](/2015/06/23/reagent-rente-install/rente-pushevent.png)

