title: "Docker開発環境 - Typesafe ActivatorでScalaをインストール"
date: 2014-06-29 00:33:28
tags:
 - DockerDevEnv
 - Scala
 - TypesafeActivator
 - LinuxMint17
description: Manningの新刊が出たのでいつものにGrokking Functional Programmingを買ってしまいました。最近Scalaでコードを書いていないので、関数型プログラミングの学習で使ってみようと思います。Scalaの複雑性の原因はsbtにあると思うのですが、どのあたりがSimple Build Toolなのか意味がわかりません。Typesafeのactivatorがその解答だと思います。今回はsbtを直接インストールしないで、Typesafe Activatorから使います。Typesafe ActivatorはWebIDEも付属していて便利ですが、単純にScalaのインストールにも使えます。
---

* `Update 2014-06-30`: [Emacs24.3とTypesafeActivatorとENSIMEでScala開発環境](/2014/06/30/emacs-scala-typesafe-activator-ensime/)


Manningの新刊が出たのでいつものように[Grokking Functional Programming](http://manning.com/khan/)を買ってしまいました。
最近Scalaでコードを書いていないので、関数型プログラミングの学習で使ってみようと思います。

Scalaの複雑性の原因はsbtにあると思うのですが、
どのあたりが`Simple Build Tool`なのか意味がわかりません。Typesafeのactivatorがその解答だと思います。

今回はsbtを直接インストールしないで、`Typesafe Activator`から使います。
`Typesafe Activator`はWebIDEも付属していて便利ですが、単純にScalaのインストールにも使えます。

### TL;DR

Dockerfileの抜粋です。

``` bash
## Java
RUN sudo add-apt-repository -y ppa:webupd8team/java \
  && sudo apt-get -y update \
  && yes | sudo apt-get install oracle-java8-installer
ENV JAVA_HOME /usr/lib/jvm/java-8-oracle

## Scala
RUN mkdir -p /home/docker/bin \
  && wget http://downloads.typesafe.com/typesafe-activator/1.2.2/typesafe-activator-1.2.2.zip -P /home/docker \
  && unzip /home/docker/typesafe-activator-1.2.2.zip -d /home/docker \
  && ln -s /home/docker/activator-1.2.2/activator /home/docker/bin/activator
```

<!-- more -->


### Dockerコンテナの起動

Dockerイメージをbuildします。

``` bash
$ docker build -t masato/baseimage:1.9 .
$ docker run --rm -i -t masato/baseimage:1.9 /sbin/my_init /bin/bash
```

### Scala


Dockerコンテナを起動します。
プロジェクトの作成

non-rootのdockerユーザーにスイッチします。
`activator new`を実行して、Scalaプロジェクトのテンプレートを作成します。


``` bash
# su - docker
$ ~/bin/activator new
Getting com.typesafe.activator activator-launcher 1.2.2 ...
...
Choose from these featured templates or enter a template name:
  1) minimal-java
  2) minimal-scala
  3) play-java
  4) play-scala
(hit tab to see a list of all templates)
> 2
Enter a name for your application (just press enter for 'minimal-scala')
>
OK, application "minimal-scala" is being created using the "minimal-scala" template.

To run "minimal-scala" from the command line, "cd minimal-scala" then:
/home/docker/minimal-scala/activator run

To run the test for "minimal-scala" from the command line, "cd minimal-scala" then:
/home/docker/minimal-scala/activator test

To run the Activator UI for "minimal-scala" from the command line, "cd minimal-scala" then:
/home/docker/minimal-scala/activator ui
```

### minimal-scala

作成した`minimal-scala`のプロジェクトをtreeで確認します。

``` bash
$ tree minimal-scala/
minimal-scala/
├── LICENSE
├── activator
├── activator-launch-1.2.2.jar
├── activator.bat
├── build.sbt
├── project
│   └── build.properties
└── src
    ├── main
    │   └── scala
    │       └── com
    │           └── example
    │               └── Hello.scala
    └── test
        └── scala
            └── HelloSpec.scala

8 directories, 8 files
```

minimal-scalaのプロジェクトに移動して、プロジェクト内の`./activator run`を実行します。
このコマンドで、sbtを実行して必要なjarをインストールと、Hello.scalaの実行を確認できます。

``` bash
$ cd minimal-scala/
$ ./activator run
Getting org.scala-sbt sbt 0.13.5 ...
...
[info] Running com.example.Hello
Hello, world!
[success] Total time: 26 s, completed 2014/06/28 14:30:27
```

### Scala REPL

ScalaのREPLも、activatorで実行できます。

``` bash
$ ./activator console
[info] Loading project definition from /home/docker/minimal-scala/project
[info] Set current project to minimal-scala (in build file:/home/docker/minimal-scala/)
[info] Starting scala interpreter...
[info]
Welcome to Scala version 2.11.1 (Java HotSpot(TM) 64-Bit Server VM, Java 1.8.0_05).
Type in expressions to have them evaluated.
Type :help for more information.

scala>
```

### Typesafe Activator UI

こちらの方が本来の使い方ですが、ブラウザから開発できるWebIDEが使ってみます。
しばらくこのコンテナを使いたいので、今度は`--rm`オプションでなく、`--name`をつけて起動します。

``` bash
$ docker run --name scaladev -p 8888 -d -t masato/baseimage:1.9 /sbin/my_init
2b8232daf65b876014c7896bcb6c7bc7bfcb8a08556fadbc3dc702e779820e19
```

IPアドレスを確認します。

``` bash
$ docker inspect 2b8232daf65b | jq -r '.[0] | .NetworkSettings | .IPAddress'
172.17.0.2
```

SSHで接続します。

``` bash
$ ssh root@172.17.0.2 -i ~/.ssh/my_key
Warning: Permanently added '172.17.0.2' (ECDSA) to the list of known hosts.
root@2b8232daf65b:~#
```

un-rootユーザーにスイッチして、`activator ui`を起動します。
コンテナで起動しているサーバーに、Dockerホストから接続できるように、0.0.0.0にバインドします。

``` bash
# su - docker
~/bin/activator ui -Dhttp.address=0.0.0.0
```

### ブラウザで確認

DockerホストのLinuxMint17からブラウザを開き、Scalaを起動しているコンテナのURLを開きます。

http://172.17.0.2:8888


### まとめ

モダンな`Typesafe Activator UI`は使っていて楽しく、チュートリアルもあって勉強になりますが、Scalaやsbtの複雑性を隠蔽できているかまだわかりません。

やはり`activator`で開発環境をつくったあとはEmacsで開発したいので、scala-mode2やENSIMEでコーディングの準備をしたいと思います。



