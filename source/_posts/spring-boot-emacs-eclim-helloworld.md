title: "EclimでSpring Boot入門 - Part1: Hello World"
date: 2014-10-05 01:01:16
tags:
 - DockerDevEnv
 - SpringBoot
 - Emacs
 - Eclim
 - Maven
description: EmacsとEclimの開発環境ができたので、Spring BootのHello Worldを作ってみます。Eclimを使うとEmacsからEclipseに接続してサブセットの機能が使えます。Eclipseと連携したMavenも実行できるのでGUI環境でなくてもJavaの開発ができます。Spring Boot入門ハンズオンをテキストにしてモダンなJavaのWebアプリを学習していこうと思います。
---

* `Update 2014-10-06`: [Docker開発環境をつくる - Emacs24とEclimでOracle Java 8のSpring Boot開発環境は失敗したのでOpenJDK 7を使う](/2014/10/06/docker-devenv-emacs24-eclim-java8-failed-spring-boot/)

[EmacsとEclim](/2014/10/04/docker-devenv-emacs24-eclim-java/)の開発環境ができたので、`Spring Boot`の`Hello World`を作ってみます。
Eclimを使うとEmacsからEclipseに接続してサブセットの機能が使えます。Eclipseと連携したMavenも実行できるのでGUI環境でなくてもJavaの開発ができます。
[Spring Boot入門ハンズオン](http://www.slideshare.net/makingx/grails-30-spring-boot)をテキストにしてモダンなJavaのWebアプリを学習していこうと思います。

<!-- more -->

### Docker開発環境

[JavaのDocker開発環境](/2014/10/04/docker-devenv-emacs24-eclim-java/)にSSH接続します。

``` bash
$ docker run --name eclim -d -i -t masato/eclim
$ docker inspect --format="&#123;&#123; .NetworkSettings.IPAddress }}" eclim
172.17.0.211
$ ssh docker@172.17.0.211
```

### eclimd を起動

コンテナに接続したら、eclimdを起動します。

``` bash
$ DISPLAY=:1 ./eclipse/eclimd -b
```

### Mavenアーキタイプ

``` bash
$ mvn -B archetype:generate \
 -DgroupId=com.example \
 -DartifactId=jsug-helloworld \
 -Dversion=1.0.0-SNAPSHOT \
 -DarchetypeArtifactId=maven-archetype-quickstart
```

### Eclipseの設定ファイルを作成

Eclipseと連携するための設定ファイルを作成します。

``` bash
$ cd jsug-helloworld
$ mvn eclipse:eclipse
```

### emacs-eclim

Emacsを起動してMavenアーキタイプで作成したプロジェクトをインポートします。

``` bash
$ emacs
M-x eclim-project-import
Project Directory: ~/jsug-helloworld/
Imported project 'jsug-helloworld'.
M-x eclim-project-mode
  | open   | jsug-helloworld                | /home/docker/jsug-helloworld
```

### pom.xml

pom.xmlを編集します。parent,dependencies,pluginsを追記します。

``` xml ~/jsug-helloworld/pom.xml
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/maven-v4_0_0.xsd">
  <modelVersion>4.0.0</modelVersion>
  <groupId>com.example</groupId>
  <artifactId>jsug-helloworld</artifactId>
  <packaging>jar</packaging>
  <version>1.0.0-SNAPSHOT</version>
  <name>jsug-helloworld</name>
  <url>http://maven.apache.org</url>
  <parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>1.1.7.RELEASE</version>
  </parent>
  <dependencies>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-test</artifactId>
      <scope>test</scope>
    </dependency>
  </dependencies>
  <build>
    <plugins>
      <plugin>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-maven-plugin</artifactId>
      </plugin>
    </plugins>
  </build>
  <properties>
    <java.version>1.7</java.version>
  </properties>
</project>
```

Mavenのtestゴールを実行します。

``` bash
M-x eclim-maven-run
Goal: test
Tests run: 1, Failures: 0, Errors: 0, Skipped: 0
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time: 1.979s
[INFO] Finished at: Sat Oct 04 10:55:35 UTC 2014
[INFO] Final Memory: 11M/216M
[INFO] ------------------------------------------------------------------------
Compilation finished at Sat Oct  4 19:55:35
```

### App.java

App.javaを編集してmainメソッドでSpringApplicationを実行できるようにします。

``` java ~/jsug-helloworld/src/main/java/com/example/App.java
package com.example;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.EnableAutoConfiguration;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
@EnableAutoConfiguration
public class App
{
    @RequestMapping("/")
    String home() {
        return "Hello World!";
    }
    public static void main( String[] args )
    {
        SpringApplication.run(App.class,args);
    }
}
```

### mvn spring-boot:run

Mavenの`spring-boot:run`ゴールを実行します。Eclimからだと見づらくなるので、シェルから実行します。組み込みのTomcatが起動しました。

``` bash
$ mvn spring-boot:run
...
  .   ____          _            __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
 \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |_\__, | / / / /
 =========|_|==============|___/=/_/_/_/
 :: Spring Boot ::        (v1.1.7.RELEASE)
...
2014-10-04 12:37:38.339  INFO 3824 --- [           main] s.b.c.e.t.TomcatEmbeddedServletContainer : Tomcat started on port(s): 8080/http
2014-10-04 12:37:38.341  INFO 3824 --- [           main] com.example.App                          : Started App in 3.215 seconds (JVM running for 3.543)
```

### 確認

DockerホストからコンテナのTomcatに接続して確認します。

``` bash
$ curl 172.17.0.211:8080
Hello World!
```
