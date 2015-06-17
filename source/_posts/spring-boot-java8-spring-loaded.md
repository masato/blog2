title: "Java8とEmacsでSpring Boot - Part2: Spring Loadedを使いHot code reload"
date: 2014-12-06 13:13:56
tags:
 - SpringBoot
 - Emacs
 - Java8
description: ここ数年はRailsやSinatraを使った開発が多くなったので、開発環境ではアプリケーションサーバー再起動しくてもコードの編集が反映されるのが当たり前のようになっています。Javaはコンパイルが必要なので、classファイルのhot reloadにはEclipseのプラグインなどを使わないと実現が難しかったと思います。RailsでEmacsを使った開発環境に慣れてしまうと、もうEclipseには戻れないので良い方法を探しています。
---

ここ数年はRailsやSinatraを使った開発が多くなったので、開発環境ではアプリケーションサーバー再起動しくてもコードの編集が反映されるのが当たり前のようになっています。Javaはコンパイルが必要なので、classファイルのhot reloadにはEclipseのプラグインなどを使わないと実現が難しかったと思います。RailsでEmacsを使った開発環境に慣れてしまうと、もうEclipseには戻れないので良い方法を探しています。

<!-- more -->

### Spring Loaded

ドキュメントの[67. Hot swapping](http://docs.spring.io/spring-boot/docs/current/reference/html/howto-hotswapping.html)に、[Spring Loaded](https://github.com/spring-projects/spring-loaded)を使ったHot code reloadの方法が書いてありました。 

STS(Eclipse)やIntelliJなどのIDEは、ソースコードの保存と同時にコンパイルをしてくれるので、classファイルの再作成と置換をツールが面倒をみてくれます。

* [67.6.1 Configuring Spring Loaded for use with Gradle and IntelliJ](http://docs.spring.io/spring-boot/docs/current/reference/html/howto-hotswapping.html#howto-reload-springloaded-gradle-and-intellij)
* [Hot Swapping in Spring Boot with Eclipse STS](http://blog.netgloo.com/2014/05/21/hot-swapping-in-spring-boot-with-eclipse-sts/)


### Emacsの場合

build.gradleのbuildscript > dependenciesに、springloadedのクラスパスを追加します。

``` groovy ~/helloworld/build.gradle
buildscript {
...
    dependencies {
        classpath("org.springframework.boot:spring-boot-gradle-plugin:${springBootVersion}")
        classpath("org.springframework:springloaded:1.2.1.RELEASE")
    }
```

gradleのbootRunタスクを実行して、組み込みのTomcatを使いアプリを起動します。

``` bash
$ ./gradlew bootRun
```

Tomcatが起動中にJavaのソースコードを編集しても自動的にコンパイルしてくれないので、適当なタイミングで手動でコマンドを実行します。bootRunを実行したままでhot reloadができるので、Tomcatの再起動は不要です。

``` bash
$ ./gradlew compileJava
```

