title: "Oracle Java 8とauto-java-completeでSpringBootの開発環境を作る"
date: 2014-12-04 21:31:57
tags:
 - DockerDevEnv
 - Emacs
 - Cask
 - auto-complete
 - auto-java-complete
 - Java
 - Java8
 - Eclim
 - SpringBoot
description: 2ヶ月前、Emacs24とEclimでOracle Java 8のインストールは失敗してしまいました。Spring BootをOracle Java 8で開発するために、Eclimをやめてauto-java-completeを使うように変更します。作成したbaseimageはDocker Hub Registryに Automated Buildしています。

---

2ヶ月前、[Emacs24とEclimでOracle Java 8](/2014/10/06/docker-devenv-emacs24-eclim-java8-failed-spring-boot/)のインストールは失敗してしまいました。Spring BootをOracle Java 8で開発するために、Eclimをやめて[auto-java-complete](https://github.com/emacs-java/auto-java-complete)を使うように変更します。作成したbaseimageは[Docker Hub Registry](https://registry.hub.docker.com/u/masato/baseimage/)に Automated Buildしています。

<!-- more -->

### Dockerfile

OpenJDK 7からOracle Java 8に変更します。

``` bash  ~/docker_apps/baseimage/Dockerfile
## apt-get update
RUN add-apt-repository ppa:webupd8team/java && \
    echo oracle-java8-installer shared/accepted-oracle-license-v1-1 select true | /usr/bin/debconf-set-selections
RUN apt-get update && \
    DEBIAN_FRONTEND=noninteractive \
    apt-get install -y build-essential software-properties-common \
                       zlib1g-dev libssl-dev libreadline-dev libyaml-dev \
                       libxml2-dev libxslt-dev sqlite3 libsqlite3-dev \
                       emacs24-nox emacs24-el git byobu wget curl unzip tree \
                       python-dev python-pip golang nodejs npm \
                       oracle-java8-installer ant maven
ENV JAVA_HOME /usr/lib/jvm/java-8-oracle
```

### auto-complete-java

CaskからEclimを削除します。auto-complete-javaのインストールには　auto-complete と [YASnippet](https://github.com/capitaomorte/yasnippet)　が必要なのでCaskに追加します。

``` bash ~/docker_apps/baseimage/dotfiles/.emacs.d/Cask
...
(depends-on "auto-complete")
...
;; groovy
(depends-on "groovy-mode")

;; java
(depends-on "helm")
(depends-on "yasnippet")
```

[auto-java-complete](https://github.com/emacs-java/auto-java-complete)はCaskからインストールできないようなので、git clone します。また、Tags.javaの中にASCII文字以外が入っているので、encodingは明示的にUTF-8にします。

``` bash  ~/docker_apps/baseimage/Dockerfile
## auto-java-complete
RUN cd ${HOME}/.emacs.d && \
    git clone https://github.com/emacs-java/auto-java-complete && \
    cd auto-java-complete && \
    javac Tags.java -encoding UTF-8 && \
    java -cp "/usr/lib/jvm/java-8-oracle/jre/lib/rt.jar:." Tags
```

auto-completeの設定をします。

``` el  ~/docker_apps/baseimage/dotfiles/.emacs.d/inits/04-auto-complete.el
(require 'auto-complete-config)
(require 'auto-complete)
(global-auto-complete-mode t)
(add-to-list 'ac-dictionary-directories "~/.emacs.d/auto-complete/ac-dict")
(ac-config-default)
(setq ac-use-menu-map t)
(setq ac-delay 0.1)
(setq ac-auto-show-menu 0.2)
```

auto-java-completeはgit cloneしたディレクトリをload-pathに追加します。

``` el  ~/docker_apps/baseimage/dotfiles/.emacs.d/inits/07-auto-java-complete.el
(add-to-list 'load-path "~/.emacs.d/auto-java-complete")
(require 'ajc-java-complete-config)
(add-hook 'java-mode-hook 'ajc-java-complete-mode)
```

[YASnippet](https://github.com/capitaomorte/yasnippet)をrequireします。

``` el  ~/docker_apps/baseimage/dotfiles/.emacs.d/inits/08-yasnippet.el
(require 'yasnippet)
(yas-global-mode 1)
```

ついでに[Helm](https://github.com/emacs-helm/helm)をインストールします。`C-x C-f`したときにDiredよりも便利に使えます。

``` el  ~/docker_apps/baseimage/dotfiles/.emacs.d/inits/09-helm.el
(require 'helm-config)
(helm-mode 1)
```