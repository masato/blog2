title: "Dockerで開発環境をつくる - Clojureのインストール"
date: 2014-09-18 00:07:30
tags:
 - DockerDevEnv
 - StreamProcessing
 - Clojure
 - Leiningen
 - Riemann
description: StreamProcessingでAlertingを出すために、Riemannを使い始めました。最近のインフラはDocker上で構築するようにしているので監視対象ホストもDockerコンテナです。StreamProcessingを記述する場合は、ClojureやScalaの関数型言語が適切です。The New Stack Makers Adrian Cockcroft on Sun, Netflix, Clojure, Go, Docker and Moreに、A lot of the best programmers and the most productive programmers I know are writing everything in Clojure and swearing by it, and then just producing ridiculously sophisticated things in a very short time.とコメントがあります。最近は会社でも10倍の生産性が話題ですが、Productive ProgrammerになるにはClojureもマスターしないと。
---

StreamProcessingでAlertingを出すために、Riemannを使い始めました。最近のインフラはDocker上で構築するようにしているので監視対象ホストもDockerコンテナです。StreamProcessingを記述する場合は、ClojureやScalaの関数型言語が適切です。
[The New Stack Makers: Adrian Cockcroft on Sun, Netflix, Clojure, Go, Docker and More](http://thenewstack.io/the-new-stack-makers-adrian-cockcroft-on-sun-netflix-clojure-go-docker-and-more/)に、
> A lot of the best programmers and the most productive programmers I know are writing everything in Clojure and swearing by it, and then just producing ridiculously sophisticated things in a very short time.

とコメントがあります。最近は会社でも[10倍の生産性](http://d.hatena.ne.jp/takeda25/20140222/1393072150)が話題ですが、`Productive Programmer`になるにはClojureもマスターしないと。

<!-- more -->

### Dockerオフィシャルのclojure

オフィシャルイメージの[clojure](https://registry.hub.docker.com/_/clojure/)を使います。
GitHubのリポジトリは、[Quantisan/docker-clojure](https://github.com/Quantisan/docker-clojure)です。

プロジェクトを作成します。

``` bash
$ mkdir ~/docker_apps/clojure
$ cd !$
```

ビルドするだけのDockerfileを作成します。

``` bash ~/docker_apps/clojure/Dockerfile
FROM clojure
WORKDIR /root
```

イメージをビルドします。

``` bash
$ docker build -t masato/clojure .
```

コンテナを起動します。

``` bash
$ docker run -it --rm --name clojuredev masato/clojure /bin/bash
root@83f5b6b8748d:/root# ls
```

### Leiningen

[Leiningen](http://leiningen.org/)の確認です。
`Leiningen 2.4.3`が入っています。

``` bash
$ which lein
/usr/local/bin/lein
$ lein --version
Leiningen 2.4.3 on Java 1.7.0_65 OpenJDK 64-Bit Server VM
```

REPLでClojureのバージョンを確認します。
`Clojure 1.6.0`が入っています。

``` bash
$ lein new hello-world
Generating a project called hello-world based on the 'default' template.
The default template is intended for library projects, not applications.
To see other templates (app, plugin, etc), try `lein help new`.
$ cd hello-world
$ lein repl
(Retrieving org/clojure/clojure/1.6.0/clojure-1.6.0.pom from central)
(Retrieving org/clojure/clojure/1.6.0/clojure-1.6.0.jar from central)
nREPL server started on port 45413 on host 127.0.0.1 - nrepl://127.0.0.1:45413
REPL-y 0.3.2, nREPL 0.2.3
Clojure 1.6.0
OpenJDK 64-Bit Server VM 1.7.0_65-b32
    Docs: (doc function-name-here)
          (find-doc "part-of-name-here")
  Source: (source function-name-here)
 Javadoc: (javadoc java-object-or-class-here)
    Exit: Control+D or (exit) or (quit)
 Results: Stored in vars *1, *2, *3, an exception in *e

user=> (clojure-version)
"1.6.0"
```

### 今後の方針

データ分析の環境もそろそろリアルタイムストリーム分析を考えないといけないです。
とりあえずEmacsで開発環境をつくってから、RiemannとLaminaを試してみようと思います。
