title: "GroovyでSpring Boot - Part1: Spring Boot CLIをGVMでインストールする"
date: 2014-11-30 23:00:42
tags:
 - SpringBoot
 - Groovy
 - GVM
description: PaktからLerning Spring Bootを購入して本格的にSpring Bootを勉強していきます。Eclim環境を作ったときはMavenでインストールしましたが、今回はGVM経由でインストールしてみます。
---

Paktから[Lerning Spring Boot](https://www.packtpub.com/application-development/learning-spring-boot)を購入して本格的にSpring Bootを勉強していきます。[Eclim環境](/2014/10/05/spring-boot-emacs-eclim-helloworld/)を作ったときはMavenでインストールしましたが、今回はGVM経由でインストールしてみます。

<!-- more -->

### GVMのインストール

[GVM](http://gvmtool.net/)をワンライナーでインストールします。GVMは、Groovy enVironment Managerの略です。RubyのrbenvみないなGroovyのバージョン管理ツールです。

``` bash
$ curl -s get.gvmtool.net | bash
```

### Spring Boot's CLIのインストール

GVMを使ってSpring Boot' CLIをインストールします。GVMを使うとSpring Boot以外にもVert.xやGrailsなどGroovyをつかった環境が簡単にインストールできます。Vert.xも余裕が出てきたので使っていきたいです。

GVMでSpring Boot's CLIをインストールします。

``` bash
$ gvm install springboot
```

今回インストールしたSpringのバージョンを確認します。

``` bash
$ spring --version
Spring CLI v1.1.9.RELEASE
```

インストール可能はバージョンをリストします。

``` bash
$ gvm ls springboot

================================================================================
Available Springboot Versions
================================================================================
     1.2.0.RC2
 > * 1.1.9.RELEASE
     1.1.8.RELEASE
     1.1.7.RELEASE
     1.1.6.RELEASE
     1.1.5.RELEASE
     1.1.4.RELEASE
     1.1.3.RELEASE
     1.1.2.RELEASE
     1.1.1.RELEASE
     1.1.0.RELEASE
     1.0.2.RELEASE
     1.0.1.RELEASE
     1.0.0.RELEASE


================================================================================
+ - local version
* - installed
> - currently in use
================================================================================
```
