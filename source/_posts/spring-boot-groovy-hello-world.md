title: "GroovyでSpring Boot - Part3: Hello World"
date: 2014-12-03 23:25:01
tags:
 - SpringBoot
 - Groovy
description: Learning Spring Bootを読みながら、まずはお約束のHello Worldから。
---


[Learning Spring Boot](https://www.packtpub.com/application-development/learning-spring-boot)を読みながら、まずはお約束のHello Worldから。


単純にクラスを定義するたけです。Javaで書くときよりもimportを省略できるのですっきりします。これだとRailsみたいに、Emacsで開発もできます。Flaskだと思えばアノテーションも気にならないし。

``` groovy app.grooy
@RestController
class App {
  @RequestMapping("/")
  def home() {
    "Hello, world!"
  }
}
```

アプリの起動は、spring runで行います。組み込みのTomcatが起動するので、アプリケーションサーバーにデプロイしなくても確認ができます。

``` bash
$ spring run app.groovy
```

今回は簡単にcurlからアプリの起動を確認してみます。とてもかんたん。

``` bash
$ curl localhost:8080
Hello, world!
```
