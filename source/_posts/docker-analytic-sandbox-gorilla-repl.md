title: "Dockerデータ分析環境 - Part8: Gorilla REPLとCMDのcdでno such file or directoryになる場合"
date: 2014-09-29 09:34:56
tags:
 - AnalyticSandbox
 - Clojure
 - GorillaREPL
 - IPythonNotebook
 - Incanter
 - REPL
description: ClojureのDocker開発環境はすでに作成していたので、Gorilla REPLもleinにプラグインを設定するだけだと思っていたのですが、意外なところで嵌まってしまいました。DockerfileのCMDの書式の違いで、Linuxのcdコマンドがexecutableでないことが原因でした。Gorilla REPLは見た目もIPythonNotebookに似ていて使いやすいです。Incanterと統合できるプラグインもあるので少しずつ拡張していきます。
---

[ClojureのDocker開発環境](/2014/09/20/docker-devenv-emacs24-clojure/)はすでに作成していたので、`Gorilla REPL`もleinにプラグインを設定するだけだと思っていたのですが、意外なところで嵌まってしまいました。
DockerfileのCMDの書式の違いで、Linuxのcdコマンドがexecutableでないことが原因でした。

`Gorilla REPL`は見た目もIPythonNotebookに似ていて使いやすいです。Incanterと統合できる[プラグイン](https://github.com/JonyEpsilon/incanter-gorilla)もあるので少しずつ拡張していきます。

<!-- more -->

### Dockerfile

プロジェクトを作成します。

``` bash
$ mkdir ~/docker_apps/gorilla
$ cd !$
```

最初に完成したDockerfileです。

`lein gorilla`でgorillaを起動します。ipアドレスを0.0.0.0にバインドして、`Gorilla REPL`にDockerコンテナの外部から接続可能にします。またデフォルトだとportはランダムなので8080を明示的に指定します。

``` bash
FROM ubuntu:14.04
MAINTAINER Masato Shimizu <ma6ato@gmail.com>

ENV HOME /root

## apt-get update
RUN sed -i~ -e 's/archive.ubuntu.com/ftp.jaist.ac.jp/' /etc/apt/sources.list \
 && apt-get update

## Japanese Environment
RUN DEBIAN_FRONTEND=noninteractive apt-get install -y language-pack-ja
ENV LANG ja_JP.UTF-8
RUN update-locale LANG=ja_JP.UTF-8
RUN mv /etc/localtime /etc/localtime.org
RUN ln -s /usr/share/zoneinfo/Asia/Tokyo /etc/localtime

RUN DEBIAN_FRONTEND=noninteractive \
    apt-get install -y build-essential software-properties-common \
                       zlib1g-dev libssl-dev libreadline-dev libyaml-dev \
                       libxml2-dev libxslt-dev sqlite3 libsqlite3-dev \
                       emacs24-nox emacs24-el git byobu wget curl unzip tree\
                       python
WORKDIR /root

# Cask
RUN curl -fsSkL https://raw.github.com/cask/cask/master/go | python
ADD .emacs.d /root/.emacs.d/
RUN echo 'export PATH="/root/.cask/bin:$PATH"' >> /root/.profile && \
    /bin/bash -c 'source /root/.profile && cd /root/.emacs.d && cask install'
ENV PATH /root/.cask/bin:$PATH

# Clojure
ADD .lein/profiles.clj /root/.lein/
RUN DEBIAN_FRONTEND=noninteractive \
 && apt-get install -y openjdk-7-jdk
RUN curl -s https://raw.githubusercontent.com/technomancy/leiningen/2.5.0/bin/l\
ein > \
    /usr/local/bin/lein && \
    chmod 0755 /usr/local/bin/lein
ENV LEIN_ROOT 1
RUN lein

# Gorilla
RUN cd /root && lein new gorilla \
 && sed -i~ -e 's/)$//' -e '$s/$/\n  :plugins [[lein-gorilla "0.3.3"]])/' goril\
la/project.clj \
 && cd gorilla && lein

CMD cd /root/gorilla && /usr/local/bin/lein gorilla :ip 0.0.0.0 :port 8080
#CMD ["cd /root/gorilla && /usr/local/bin/lein gorilla :ip 0.0.0.0 :port 8080"]

# Clean
RUN apt-get clean && rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*
```

### CMDでcdを使うとno such file or directoryになる場合

[CMDのドキュメント](https://docs.docker.com/reference/builder/#cmd)には以下のように書いてあります。

> CMD ["executable","param1","param2"] (like an exec, this is the preferred form)
CMD ["param1","param2"] (as default parameters to ENTRYPOINT)
CMD command param1 param2 (as a shell)

cdコマンドはシェルの組み込みコマンドなので、ファイル形式のコマンドではありません。

``` bash
$ type cd
cd はシェル組み込み関数です
```

[Location of cd executable](http://stackoverflow.com/questions/3741148/location-of-cd-executable)を読むまでcdコマンドにはバイナリがあると思っていました。以下のように`JSON Array`でCMDの最初でcdをするとエラーになります。

``` bash
CMD ["cd /root/gorilla && /usr/local/bin/lein gorilla :ip 0.0.0.0 :port 8080"]
```

コンテナを起動すると`no such file or directory`が発生します。

``` bash
$ docker run -it --rm masato/gorilla
2014/09/29 10:21:24 Error response from daemon: Cannot start container 49b251c65546b2e83aa0cf2a9df1dce52c99c8b5a2a33f2edcbfb0c590a6b387: exec: "cd /root/gorilla && /usr/local/bin/lein gorilla :ip 0.0.0.0 :port 8080": stat cd /root/gorilla && /usr/local/bin/lein gorilla :ip 0.0.0.0 :port 8080: no such file or directory
```

cdをしながらコマンドを実行する場合は、以下のように`curly braces`を使わずにシェルで実行します。

``` bash
CMD cd /root/gorilla && /usr/local/bin/lein gorilla :ip 0.0.0.0 :port 8080
```

### build && run

イメージをビルドしてDockerコンテナを起動します。指定した8080ポートで起動しました。

```
$ docker build -t masato/gorilla .
$ docker run -it --rm  masato/gorilla
Retrieving org/clojure/tools.nrepl/0.2.6/tools.nrepl-0.2.6.pom from central
Retrieving clojure-complete/clojure-complete/0.2.3/clojure-complete-0.2.3.pom from clojars
Retrieving org/clojure/tools.nrepl/0.2.6/tools.nrepl-0.2.6.jar from central
Retrieving clojure-complete/clojure-complete/0.2.3/clojure-complete-0.2.3.jar from clojars
Gorilla-REPL: 0.3.3
Started nREPL server on port 43110
Running at http://localhost:8080/worksheet.html .
Ctrl+C to exit.
```

### 確認

GorillaコンテナのIPアドレスを確認します。

``` bash
$ docker inspect -format="&#123;&#123; .NetworkSettings.IPAddress }}" 5bc5a5a76a30
Warning: '-format' is deprecated, it will be replaced by '--format' soon. See usage.
172.17.2.3
```

ブラウザで確認します。

{% img center /2014/09/29/docker-analytic-sandbox-gorilla-repl/gorilla-repl.png %}


