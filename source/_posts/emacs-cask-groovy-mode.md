title: "GroovyでSpring Boot - Part2: baseimageにGVMとgroovy-modeを追加する"
date: 2014-12-01 21:55:02
tags:
 - DockerDevEnv
 - Groovy
 - SpringBoot
description: Spring BootをGroovyで開発するため、Docker開発環境のbaseimageにGVMとEmacsのgroovy-modeをインストールします。
---

Spring BootをGroovyで開発するため、Docker開発環境の[baseimage](https://registry.hub.docker.com/u/masato/baseimage/)にGVMとEmacsのgroovy-modeをインストールします。

<!-- more -->

### groovy-mode

Caskにgroovy-modeを追加します。

``` bash ~/docker_apps/baseimage/dotfiles/.emacs.d/Cask
;; groovy
(depends-on "groovy-mode")
```

### baseimage/Dockerfile

DockerfileにGroovyとSpring Bootのインストールを追加します。

``` bash ~/docker_apps/baseimage/Dockerfile
## Groovy/Spring Boot
RUN curl -s get.gvmtool.net | bash && \
    /bin/bash -c 'source ${HOME}/.gvm/bin/gvm-init.sh && gvm install springboot'
```

