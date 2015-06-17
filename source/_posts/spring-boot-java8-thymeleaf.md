title: "Java8とEmacsでSpring Boot - Part3: Thymeleafテンプレートエンジン"
date: 2014-12-07 15:45:42
tags:
 - SpringBoot
 - Emacs
 - Thymeleaf
 - Java8
description: Hello Worldで作成した@RestControllerを修正して、ビューのレンダリングにThymeleafテンプレートエンジンを利用していきます。通常Spring MVCアプリを作るときは最初からMVCでクラスを分けて書いていきますが、Spring Bootを使うと必要な機能を少しずつ追加してもよい、お手軽な感じがします。
---

[Hello World](/2014/12/05/spring-boot-emacs-java8-hello-world/)で作成した@RestControllerを修正して、ビューのレンダリングにThymeleafテンプレートエンジンを利用していきます。通常Spring MVCアプリを作るときは最初からMVCでクラスを分けて書いていきますが、Spring Bootを使うと必要な機能を少しずつ追加してもよい、お手軽な感じがします。

<!-- more -->

### Spring Boot starters

Hello Worldアプリは[start.spring.io](http://start.spring.io/)のフォームを使い、Template EnginesはにThymeleafにチェックを入れてBootstrapを作成しているので、build.gradleにはspring-boot-starter-thymeleafのstarterがdependenciesがすでに入っています。

``` groovy ~/helloworld/build.gradle
dependencies {
    compile("org.springframework.boot:spring-boot-starter-thymeleaf")
    testCompile("org.springframework.boot:spring-boot-starter-test")
}
```

spring-boot-starter-thymeleafの[pom.xml](https://github.com/spring-projects/spring-boot/blob/master/spring-boot-starters/spring-boot-starter-thymeleaf/pom.xml)を見ると、spring-boot-starterやspring-boot-starter-webのstarterも入っています。このstarterを使うと、組み込みTomcat、JSON、Spring MVCとThymeleafを使った一般敵なWebアプリがすぐに作れる状態になっています。

### @Controller

今回は以下のアノテーションを使います。

* @RequestMapping: HTTPリクエストにマッピングする
* @RequestParam: リクエストパラメーターを受け取る

"/"へのGETのパラメーターとしてnameを受け取ります。パラメーターがなかった場合は"World"をデフォルト値として使います。引数のModelにビューでレンダリングする値としてnameパラメーターをセットします。

``` java ~/helloworld/src/main/java/helloworld/Application.java
package helloworld;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.EnableAutoConfiguration;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestMethod;
import org.springframework.stereotype.Controller;
import org.springframework.ui.Model;

@Controller
@EnableAutoConfiguration
public class Application {

    @RequestMapping(value = "/", method = RequestMethod.GET)
    public String home(@RequestParam(value="name", defaultValue="World") String name,
                       Model model) {
        model.addAttribute("name", name);
        return "home/index";
    }

    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

### Thymeleafテンプレート

テンプレートを配置するディレクトリを作成します。Thymeleafで使うテンプレートは純粋なHTMLファイルなので、JSPと違いそのままブラウザでも表示できます。

``` bash
$ mkdir ~/helloworld/src/main/resources/templates/
```

homeメソッドの戻り値、`"home/index"` がテンプレートのパスになるため、homeディレクトリとindex.htmlファイルを作成します。


``` html ~/helloworld/src/main/resources/templates/home/index.html
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org">
  <head>
    <title>Hello World</title>
  </head>
  <body>
    <p th:text="'Hello, ' + ${name}"></p>
  </body>
</html>
```

### 確認

アプリケーションサーバーを起動します。

``` bash
$ cd ~/helloworld
$ ./gradlew bootRun
```

curlコマンドを使い、最初は引数なしでサーバーにリクエストを送信します。デフォルト値の"World"が表示されました。

``` bash
$ curl localhost:8080
<!DOCTYPE html>

<html>
  <head>
    <title>Hello World</title>
  </head>
  <body>
    <p>Hello, World</p>
  </body>
</html>
```

今度はクエリ文字列にパラメーターを指定してリクエストを送信します。パラメーターの値がレンダリングされました。

``` bash
$ curl localhost:8080?name=Masato
<!DOCTYPE html>

<html>
  <head>
    <title>Hello World</title>
  </head>
  <body>
    <p>Hello, Masato</p>
  </body>
</html>
```