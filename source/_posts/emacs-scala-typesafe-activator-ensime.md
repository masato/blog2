title: "Emacs24.3とTypesafeActivatorとENSIMEでScala開発環境"
date: 2014-06-30 00:54:17
tags:
 - TypesafeActivator
 - Emacs
 - Scala
 - ENSIME
 - Java
description: GitHubのensimeのリポジトリはもうすぐDEPRECATEDになるようで、ensime-serverに移動したようです。Emacsのパッケージはensime-emacsになりました。Sublimeからも使われるようになったからでしょうか。Scalaやsbtのバージョン管理やディレクトリ構造は私には複雑すぎです。Scalaな人たちは普通なのでしょうが、どうもこの雰囲気に馴染めません。
---

* `Update 2014-10-04`: [Docker開発環境をつくる - Emacs24とEclimからEclipseに接続するJava開発環境](/2014/10/04/docker-devenv-emacs24-eclim-java/)

GitHubの[ensime](https://github.com/ensime/ensime)のリポジトリはもうすぐDEPRECATEDになるようで、[ensime-server](https://github.com/ensime/ensime-server)に移動したようです。
Emacsのパッケージは[ensime-emacs](https://github.com/ensime/ensime-emacs)になりました。Sublimeからも使われるようになったからでしょうか。

Scalaやsbtのバージョン管理やディレクトリ構造は、私には複雑すぎです。
Scalaな人たちは普通なのでしょうが、どうもこの雰囲気に馴染めません。

### TL;DR

`Typesafe Activator`でインストールしたScalaとsbtも、ENSIME-Emacsから使うことができましたが、
2014-06-30は、うってつけの日ではなかったようです。

<!-- more -->


### scala-mode2のインストール

Emacsを起動して、`M-x list-packages`を実行、
scala-mode2を検索、選択(i)して、実行(x)します。package.elは便利！

### 謎のENSIME配布パッケージ

packege.elの`M-x list-packages`で表示されるMELPAの[ensime](http://melpa.milkbox.net/#/ensime)は、
[ensime-20131030.1503.tar](
http://melpa.milkbox.net/packages/ensime-20131030.1503.tar)です。

GitHubのリンクは[DEPRECATED](https://github.com/ensime/ensime)の方で2013-10-30と更新が古いです。

一方、新しいensime-serverリポジトリの[README](https://github.com/ensime/ensime-server)からリンクがある[ENSIME Releases](https://www.dropbox.com/sh/ryd981hq08swyqr/V9o9rDvxkS/ENSIME%20Releases)の最新は、
[ensime_2.10.0-0.9.8.9.tar.gz](
https://www.dropbox.com/sh/ryd981hq08swyqr/AAAdLAGm5no1XDLimVbAXE9Za/ENSIME%20Releases/ensime_2.10.0-0.9.8.9.tar.gz)で、Modifiedが1年前です。(日付はいいつなの？)

また、[ENSIME User Manual](http://ensime.github.io/)に書いてある[distributionページ](https://github.com/ensime/ensime-server/downloads)の最新は、
[ensime_2.10.0-RC3-0.9.8.2.tar.gz](https://github.com/downloads/ensime/ensime-server/ensime_2.10.0-RC3-0.9.8.2.tar.gz)で、更新は`Uploaded on 2 Dec 2012`です。

3種類の配布パッケージがあって、この時点ですでに心が折れているのですが、がんばります。

更新日付だけ見ると、MELPAが一番新しいようですが、そもそもこの世界では更新日付は意味がないのかも知れません。

* [ensime-20131030.1503.tar](
http://melpa.milkbox.net/packages/ensime-20131030.1503.tar)
* [ensime_2.10.0-0.9.8.9.tar.gz](
https://www.dropbox.com/sh/ryd981hq08swyqr/AAAdLAGm5no1XDLimVbAXE9Za/ENSIME%20Releases/ensime_2.10.0-0.9.8.9.tar.gz)
* [ensime_2.10.0-RC3-0.9.8.2.tar.gz](https://github.com/downloads/ensime/ensime-server/ensime_2.10.0-RC3-0.9.8.2.tar.gz)

ファイルやパッケージのバージョンや更新日はものすごく気にする方なので、こういう状態はちょっと混乱してしまいます。

MELPAのensime-20131030.1503.tarをダウンロードして解凍してみると、libにあるのは、`ensime_2.10.0-0.9.9.jar`でした。

### MELPAのENSIMEを使う

命名規則から読み取ると、MELPAの`ensime_2.10.0-0.9.9.jar`が2014-06-30の時点で最新ぽいので、MEPLAからインストールします。

`M-x list-packages`を表示し、`Ctrl-s`でensimeを検索、選択(i)して、実行(x)します。

インストールしたパッケージlibを確認します。

``` bash
$ ls  ~/.emacs.d/elpa/ensime-20131030.1503/lib/
asm-3.3.jar            org.eclipse.jdt.core-3.6.0.v_A58.jar
asm-commons-3.3.jar    org.scala-refactoring_2.10.0-SNAPSHOT-0.6.1-20130201.063851-55.jar
asm-tree-3.3.jar       scala-actors-2.10.2.jar
asm-util-3.3.jar       scala-compiler.jar
critbit-0.0.4.jar      scala-library.jar
ensime_2.10-0.9.9.jar  scala-reflect-2.10.2.jar
json-simple-1.1.jar    scalariform_2.10-0.1.4.jar
lucene-core-3.5.0.jar
```

Scalaは`Typesafe Activator`で[インストール](/2014/06/29/docker-devenv-scala-typsafe-activator/)していました。
sbtコマンドは直接使えないので、`.ensime`を作成するためにsbtをラップしているactivatorコマンドを実行します。

`project/plugins.sbt`ファイルを作成して、ensime-sbtを記述します。

``` scala ~/spike-scala/project/plugins.sbt
addSbtPlugin("org.ensime" % "ensime-sbt" % "0.1.4")
```

`./activator "ensime generate"`を実行するとエラーになりました。

``` bash
$ ./activator "ensime generate"
...
[warn]  ::::::::::::::::::::::::::::::::::::::::::::::
[warn]  ::          UNRESOLVED DEPENDENCIES         ::
[warn]  ::::::::::::::::::::::::::::::::::::::::::::::
[warn]  :: org.ensime#ensime-sbt;0.1.4: not found
[warn]  ::::::::::::::::::::::::::::::::::::::::::::::
[warn]
[warn]  Note: Some unresolved dependencies have extra attributes.  Check that these dependencies exis
t with the requested attributes.
[warn]          org.ensime:ensime-sbt:0.1.4 (scalaVersion=2.10, sbtVersion=0.13)
[warn]
sbt.ResolveException: unresolved dependency: org.ensime#ensime-sbt;0.1.4: not found
```

### ensime-sbtをローカルに配置してやり直し

[Plugin version 0.1.3 not available](https://github.com/ensime/ensime-sbt/issues/21)を読むと、masterの最新バージョンは`0.1.4-SNAPSHOT`らしく、
ローカルにライブラリを配置して使うようです。

ensime-sbtを`git clone`します。

```
$ git clone https://github.com/ensime/ensime-sbt.git
$ cd ensime-sbt
$ activator publish-local
...
[info]  published ensime-sbt to /home/docker/.ivy2/local/org.ensime/ensime-sbt/scala_2.10/sbt_0.13/0.1.4-SNAPSHOT/srcs/ensime-sbt-sources.jar
[info]  published ivy to /home/docker/.ivy2/local/org.ensime/ensime-sbt/scala_2.10/sbt_0.13/0.1.4-SNAPSHOT/ivys/ivy.xml
[success] Total time: 11 s, completed 2014/06/29 12:17:25
```

ensime-sbtのVERSIONを、`0.1.4-SNAPSHOT`にします。

``` scala ~/spike-scala/project/plugins.sbt
addSbtPlugin("org.ensime" % "ensime-sbt" % "0.1.4-SNAPSHOT")
```

`.ensime`の作成をやり直します。


```
$ cd ~/spike-scala
$ ./activator "ensime generate"
...
[info] Wrote configuration to .ensime
```

### ENSIME用の.emacs

[ensime-emacs](https://github.com/ensime/ensime-emacs)の`Quick Start`を読むと、.emacsの設定方法が書いてあります。

``` el ~/.emacs.d/inits/08-ensime.el
(require 'ensime)
(add-hook 'scala-mode-hook 'ensime-scala-mode-hook)
```

### Emacsを起動してM-x ensime

Emacsを起動して`M-x ensime`をするとエラーが出ました。

```
Debugger entered: (("Error in timer" ensime-attempt-connection ((:subprojects ((:name "spike-scala" $
  (condition-case data (apply fun args) (error (debug nil (list "Error in timer" fun args data))))
  ensime-timer-call(ensime-attempt-connection (:subprojects ((:name "spike-scala" :module-name "spik$
  apply(ensime-timer-call (ensime-attempt-connection (:subprojects ((:name "spike-scala" :module-nam$
  byte-code("r\301^H\302H^H\303H\"\210)\301\207" [timer apply 5 6] 4)
  timer-event-handler([t 21424 1374 540297 0.3 ensime-timer-call (ensime-attempt-connection (:subpro$
```

まだ、がんばります。

### .elcを削除する

ensime-serverのissuesをググっていると、ちょうと該当するものがありました。
1年以上、この状態のようです。。

[ENSIME cannot be used when lisp code is byte compiled by MELPA [renamed] #310](https://github.com/ensime/ensime-server/issues/310)

原因はMELPAがelをコンパイルしてしまうのが問題らしく、.elcを削除します。

``` bash
$ cd ~/.emacs.d/elpa/ensime-20131030.1503
$ find . -name "*.elc" | xargs rm
```

再度Emacsを起動して、`M-x engime`を実行すると、ようやくENSIMEが起動しました。

```
ENSIME ready. Death to null!
```

### "Death to null!" ってなに？

[ensime.el](https://github.com/ensime/ensime-emacs/blob/master/ensime.el)によると、Scala的に励ましてくれているようです。

``` el ensime.el
(defvar ensime-words-of-encouragement
  `("Let the hacking commence!"
    "Hacks and glory await!"
    "Hack and be merry!"
    "May the source be with you!"
    "Death to null!"
    "Find closure!"
    "May the _ be with you."
    "M-x be_cool"
    "CanBuildFrom[List[Dream], Reality, List[Reality]]"
    ,(format "%s, this could be the start of a beautiful program."
	     (ensime-user-first-name)))
  "Scientifically-proven optimal words of hackerish encouragement.")
```

### まとめ

こういったScalaライブラリのバージョン管理の複雑性を隠蔽してくれるのが`Typesafe Activator`の位置づけなのだと思います。

ただ、sbtやMavenをラップしてjarを使っていることには変わりなく、Mavenの依存関係はちょっと嫌いなので、インターナルなリポジトリサーバーで全部管理しない限り、リリース後のライブラリ管理とか、ちょっとプロダクションでScalaをやる勇気はしないです。

個人でScalaのコードを書いているのはすごく楽しいのですが。







