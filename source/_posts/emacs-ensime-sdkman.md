title: "SDKMAN!とENSIMEとEmacsでScalaの開発環境構築"
date: 2017-08-09 08:45:32
tags:
 - Scala
 - SDKMAN
 - Emacs
 - ENSIME
description: EclimでJavaの開発環境構築をしました。同様にちょっとしたScalaのアプリを書きたい時にEclipseやIntelliJ IDEAを起動するのも重たいのでいつものEmacsでScalaの開発環境を構築します。
---

　[Eclim](https://github.com/emacs-eclim/emacs-eclim)で[Javaの開発環境構築](https://masato.github.io/2017/08/04/emacs-eclim-java/)をしました。同様にちょっとしたScalaのアプリを書きたい時にEclipseやIntelliJ IDEAを起動するのも重たいのでいつものEmacsでScalaの開発環境を構築します。

<!-- more -->

## 開発用の仮想マシン

　クラウド上にUbuntu 16.04.3の仮想マシンを用意してパッケージを更新しておきます。

```bash
$ sudo apt-get update && sudo apt-get dist-upgrade -y
```

## SDKMAN!

　[SDKMAN!](http://sdkman.io/)は[Gradle](https://gradle.org/)や[sbt](http://www.scala-sbt.org/)などJVM言語のSDK管理ツールです。最近ではJavaのバージョン管理もできるようになりました。

　ワンライナーでSDKMAN!をインストールします。

```
$ curl -s get.sdkman.io | /bin/bash
$ sdk version
SDKMAN 5.5.9+231
```

### Java

　SDKMAN!を使ってインストールできるJavaのバージョンを一覧します。

```
$ sdk list java
================================================================================
Available Java Versions
================================================================================
     8u141-oracle
     8u131-zulu
     7u141-zulu
     6u93-zulu
```

　OpenJDKベースの[Zulu](http://www.azul.com/downloads/zulu/)の8を使います。

```
$ sdk install java 8u131-zulu
```

　シェルを起動し直して`$JAVA_HOME`環境変数を確認します。

```
$ echo $JAVA_HOME
/home/cloud-user/.sdkman/candidates/java/current
```

### Scalaとsbt

　Scalaのインストール可能なバージョンです。

```
$ sdk list scala

================================================================================
Available Scala Versions
================================================================================
     2.12.3
     2.12.2
     2.12.1
     2.12.0
     2.11.8
     2.11.7
     2.11.6
     2.11.5
     2.11.4
     2.11.3
     2.11.2
     2.11.1
     2.11.0
```

　Scalaを最新の`2.12.3`をインストールします。

```
$ sdk install scala 2.12.3
```

　続いてsbtです。バージョンを指定しない場合は最新がインストールされます。

```
$ sdk install sbt
$ sdk current sbt

Using sbt version 0.13.15
```

## Emacs

　Emacs24とCaskをインストールします。

```bash
$ sudo apt-get install emacs24-nox emacs24-el -y
$ emacs --version
GNU Emacs 24.5.1
```

　パッケージ管理の[Cask](http://cask.github.io/)です。インストールする時にGitとPythonが必要です。

```bash
$ sudo apt-get install git python -y
$ curl -fsSL https://raw.githubusercontent.com/cask/cask/master/go | python
$ echo 'export PATH="$HOME/.cask/bin:$PATH"' >> ~/.bashrc
$ source ~/.bashrc
```

　`~/.emacs.d`ディレクトリに以下のような設定ファイルを用意します。

```
$ tree ~/.emacs.d/
/home/cloud-user/.emacs.d/
├── Cask
└── init.el
```

　init.elは[init-loader](https://github.com/emacs-jp/init-loader)で分割したい場合は[前回](https://masato.github.io/2017/08/04/emacs-eclim-java/)の設定を確認してください。

* ~/.emacs.d/init.el

```el
(require 'cask "~/.cask/cask.el")
(cask-initialize)
```

　Caskにインストールするパッケージを記述します。

* ~/.emacs.d/Cask

```el
(source gnu)
(source melpa)

(depends-on "cask")

;; ENSIME
(depends-on "ensime")
```

　`~/.emacs.d`ディレクトリに移動してパッケージをインストールします。

```bash
$ cd ~/.emacs.d
$ cask install
```

## ユーザーごとの設定

　sbtプラグインの[sbt-ensime](http://ensime.org/build_tools/sbt/)はユーザー単位にインストールします。プラグインのディレクトリがない場合は作成します。

```
$ mkdir -p ~/.sbt/0.13/plugins/
```

### plugins.sbt

　sbtプラグインの[sbt-ensime](http://ensime.org/build_tools/sbt/)をインストールします。ホームの`~/.sbt`にplugins.sbtを作成します。

* ~/.sbt/0.13/plugins/plugins.sbt

```scala
addSbtPlugin("org.ensime" % "sbt-ensime" % "1.12.14")
```

### global.sbt

　sbt-ensimeプラグインの設定を記述します。

* ~/.sbt/0.13/global.sbt

```scala
import org.ensime.EnsimeKeys._

ensimeIgnoreMissingDirectories := true
cancelable in Global := true
```

　カスタマイズの内容は以下を参考にしました。

* `scala_2.11`ディレクトリを作成しない
[ensimeConfig creates directories java and scala-2.11, which I don't need](https://stackoverflow.com/questions/41070767/ensimeconfig-creates-directories-java-and-scala-2-11-which-i-dont-need)

* `C-c`でプロセスをキャンセルする
[Cancel Proceses](http://ensime.org/build_tools/sbt/#cancel-processes)


## プロジェクトごとの設定

　簡単なsbtプロジェクトを作成してEmacsからENSIMEを利用してみます。

```
$ mkdir -p ~/scala_apps/spike && cd ~/scala_apps/spike
$ cd ~/scala_apps/spike
$ mkdir -p src/{main,test}/{java,resources,scala}
$ mkdir lib project target
```

### build.sbt

　プロジェクト定義をbuild.sbtに書きます。ScalaのバージョンはSDKMAN!でインストールした`2.12.3`から`2.11.8`に変えてみました。

* ~/scala_apps/spike/build.sbt

```scala
name := "Spike"
version := "0.1"
scalaVersion := "2.11.8"
```

### .ensime

　プロジェクトのトップディレクトリでsbtを起動します。`ensimeConfig`コマンドを実行して`.ensime`ファイルを作成します。

```
$ cd ~/scala_apps/spike
$ sbt 
> ensimeConfig
[info] ENSIME update.
[info] Updating {file:/home/cloud-user/scala_apps/spike/}spike...
[info] Resolving jline#jline;2.12.1 ...
[info] Done updating.
[info] Resolving org.scala-lang#scala-reflect;2.11.8 ...
[info] ENSIME processing spike (Spike)
```

## ENSIMEの使い方

　作成したScalaプロジェクトに簡単なHello Worldのコードを書きます。

### 起動

　`.ensime`ファイルがあるプロジェクトのトップディレクトリでEmacsを開きます。ENSIMEサーバーの起動には少し時間がかかります。

```
M-x ensime
```

　ミニバッファにメッセージが出るとサーバーの起動終了です。

```
ENSIME ready. May the source be with you, always.
```

### HelloWorld.scala

 mainメソッドを実装したScalaのコードを書きます。

```scala
 object HelloWorld {
   def main(args: Array[String]): Unit = {
     println("Hello World!")
   }
 }
```

### scala-mode

　[scala-mode](http://ensime.org/editors/emacs/scala-mode/)はENSIMEをインストールすると自動的に使えます。個別インストールも可能です。Scalaのコードを書く場合の基本的なモードです。例えばEmacsからScala REPLを起動してみます。キーバインドの意味は`control を押しながらc、v、controlを離してz`です。

```
C-c C-v z
```

　プロジェクトのbuild.sbtに定義した`2.11.8`のREPLが起動しました。

```
Welcome to Scala 2.11.8 (OpenJDK 64-Bit Server VM, Java 1.8.0_131).
Type in expressions for evaluation. Or try :help.

scala>
```

### sbt-mode

　[sbt-mode](http://ensime.org/editors/emacs/sbt-mode/)も自動的に使えます。このモードではEmacsからsbtを操作するときに使います。sbtのセッションを起動します。

```
M-x sbt-start
```

　runコマンドはプロジェクトのmainメソッドをプログラムを実行します。Hello worldが表示されました。

```
> run
[info] Running HelloWorld
Hello World!
[success] Total time: 0 s, completed Aug 9, 2017 9:07:09 AM
```