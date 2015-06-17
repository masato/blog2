title: "Java8とEmacsでSpring Boot - Part1: Hello World"
date: 2014-12-05 23:25:01
tags:
 - SpringBoot
 - Java8
 - Gradle
 - Emacs
description: Learning Spring BootはChapter2移行はGroovyからJavaになっています。先日作成したEmacsのauto-java-complete環境でHello Worldを作成していきます。Groovyで書く場合はRubyのようにEmacsで開発できそうですが、Javaの場合もEclipseやIntellijを使わずEmacsを使って快適に開発を行いたいです。
---

[Learning Spring Boot](https://www.packtpub.om/application-development/learning-spring-boot)はChapter2移行はGroovyからJavaになっています。先日作成したEmacsのauto-java-complete環境でHello Worldを作成していきます。Groovyで書く場合はRubyのようにEmacsで開発できそうですが、Javaの場合もEclipseやIntellijを使わずEmacsを使って快適に開発を行いたいです。

<!-- more -->

###  Spring Initializr(start.spring.io)

新しいプロジェクトを作成する場合、[Spring Initializr](http://start.spring.io)を使うと便利です。zipファイルをダウンロードして展開するとGradleプロジェクトのBootstrapを作成できます。

以下の情報をフォームに入力して、"Generate Project"ボタンを押すと、helloworld.zipファイルがダウンロードできます。

* Group: helloworld
* Artifact: helloworld
* Name: Hello World
* Description: Hello World
* Package Name: helloworld
* Type: Gradle Project
* Packaging: Jar
* Java Version: 1.8
* Language: Java
* Spring Boot Version: 1.1.9
* Template Engines: Thymeleaf

開発環境のコンテナを起動してzipファイルをコピーします。

``` bash
$ docker run -d -it --name spring masato/baseimage /sbin/my_init
$ IP_ADDRESS=$(docker inspect --format="&#123;&#123; .NetworkSettings.IPAddress }}" spring)
$ scp -o IdentitiesOnly=yes -i  ~/.ssh/insecure_key ~/helloworld.zip docker@$IP_ADDRESS:
```

コンテナにSSH接続します。

``` bash
$ ssh -A docker@$IP_ADDRESS -o IdentitiesOnly=yes -i ~/.ssh/insecure_key
```

bootstrapを解凍します。標準的なGradleのプロジェクトが作成できました。

``` bash
$ unzip -l helloworld.zip
Archive:  helloworld.zip
  Length      Date    Time    Name
---------  ---------- -----   ----
        0  2014-12-05 14:07   src/
        0  2014-12-05 14:07   src/main/
        0  2014-12-05 14:07   src/main/java/
        0  2014-12-05 14:07   src/main/java/helloworld/
        0  2014-12-05 14:07   src/main/resources/
        0  2014-12-05 14:07   src/main/resources/static/
        0  2014-12-05 14:07   src/main/resources/templates/
        0  2014-12-05 14:07   src/test/
        0  2014-12-05 14:07   src/test/java/
        0  2014-12-05 14:07   src/test/java/helloworld/
      949  2014-12-05 14:07   build.gradle
      458  2014-12-05 14:07   src/main/java/helloworld/Application.java
        0  2014-12-05 14:07   src/main/resources/application.properties
      482  2014-12-05 14:07   src/test/java/helloworld/ApplicationTests.java
---------                     -------
     1889                     14 files
$ unzip -d helloworld helloworld.zip
$ cd helloworld
```

###  Spring Boot starters

build.gradleを見ると、spring-boot-starter-thymeleafのdependenciesが入っています。Spring Boot startersは仮想プロジェクトで、Spring Bootが定義している依存関係のJARとバージョンをまとめてインストールできます。

``` groovy ~/helloworld/build.gradle
dependencies {
    compile("org.springframework.boot:spring-boot-starter-thymeleaf")
    testCompile("org.springframework.boot:spring-boot-starter-test")
}
```

Boot startersが生成するメインクラスは以下のようになっています。

``` java /helloworld/src/main/java/helloworld/Application.java
package helloworld;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.EnableAutoConfiguration;
import org.springframework.context.annotation.ComponentScan;
import org.springframework.context.annotation.Configuration;

@Configuration
@ComponentScan
@EnableAutoConfiguration
public class Application {

    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

簡単なHelloWorldを出力するメソッドを追加します。

``` java /helloworld/src/main/java/helloworld/Application.java
package helloworld;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.EnableAutoConfiguration;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
@EnableAutoConfiguration
public class Application {

    @RequestMapping("/")
    String home(){
        return "Hello World";
    }

    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```


### Gradleラッパー(gradlew)の作成

[前回インストール](/2014/12/01/emacs-cask-groovy-mode/)したGVMを使いgradleをインストールします。最新の2.2.1がインストールされました。

``` bash
$ gvm install gradle
$ gradle --version
...
Gradle 2.2.1
```

プロジェクトのディレクトリで、`gradle wrapper`を実行すると、gradlewコマンドが作成されます。プロジェクトに作成したGradleラッパーを経由してgradleコマンドを実行します。

``` bash
$ cd ~/helloworld
$ gradle wrapper
```

また、start.spring.ioのbootstrapで作成したbuild.gradleのtask wrapperを見るとgradleVersionは、1.12です。こちらも最新の2.2.1に変更します。

``` groovy ~/helloworld/build.gradle
task wrapper(type: Wrapper) {
    gradleVersion = '2.2.1'
}
```

./gradlewを実行すると、GVMでインストールしたgradleとは別に、$HOME/.gradle/wrapper/にGradle環境が構築されます。

``` bash
./gradlew
```

このgradlewはGitリポジトリにプロジェクトファイルとしてコミットします。新しいマシンで開発を行う場合は、git cloneしてから、gradlewを使い簡単に環境構築が行えます。

### アプリケーションの起動と確認

gradlewはgradleコマンドのラッパーなのでgradleと同様に使えます。アプリケーションを起動します。

``` bash
$ ./gradlew clean bootRun
```

別のシェルからcurlを使い、起動を確認します。

``` bash
$ curl localhost:8080
Hello Worlddocker
```