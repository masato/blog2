title: "Docker開発環境をつくる - Emacs24とEclimからEclipseに接続するJava開発環境"
date: 2014-10-04 13:42:08
tags:
 - DockerDevEnv
 - Java
 - Emacs
 - Cask
 - Eclim
 - Eclipse
 - Xvfb
description: ClojureやMicro Servicesの学習のためにJavaを使う機会が増えました。OSvを使うとJavaだけで理想のアーキテクチャができそうです。以前Javaの開発ではEclipseを使っていました。auto-completeがないとJavaの開発はつらいです。その後RubyやPythonで開発するようになり、メインの開発環境はEmacsになっています。最近では開発はクラウド上のDockerコンテナ内でプログラミングをしているので、さらにGUI環境から遠くなりました。X11 forwardingでEclipseもおもしろそうですが、Eclipseをネットワーク接続してEmacsから使う方法を試してみます。
---

* `Update 2014-10-06`: [Docker開発環境をつくる - Emacs24とEclimでOracle Java 8のSpring Boot開発環境は失敗したのでOpenJDK 7を使う](/2014/10/06/docker-devenv-emacs24-eclim-java8-failed-spring-boot/)
* `Update 2014-10-05`: [EclimでSpring Boot入門 - Part1: Hello World](/2014/10/05/spring-boot-emacs-eclim-helloworld/)


[Clojure](/2014/09/20/docker-devenv-emacs24-clojure/)や[Micro Services](/2014/10/03/micro-services-docker-osv-spring-cloud/)の学習のためにJavaを使う機会が増えました。[OSv](http://osv.io/)を使うとJavaだけで理想のアーキテクチャができそうです。

以前Javaの開発ではEclipseを使っていました。`auto-complete`がないとJavaの開発はつらいです。その後RubyやPythonで開発するようになり、メインの開発環境はEmacsになっています。最近では開発はクラウド上のDockerコンテナ内でプログラミングをしているので、さらにGUI環境から遠くなりました。

`X11 forwarding`でEclipseもおもしろそうですが、Eclipseをネットワーク接続してEmacsから使う方法を試してみます。

<!-- more -->

### プロジェクトの作成

プロジェクトを作成します。

``` bash
$ mkdir ~/docker_apps/eclim
```

今回のプロジェクトの構成です。

``` bash
$ tree ~/docker_apps/eclim/
~/docker_apps/eclim/
├── .emacs.d
│   ├── Cask
│   ├── init.el
│   └── inits
│       ├── 00-keybinds.el
│       ├── 01-files.el
│       └── 04-eclim.el
├── Dockerfile
├── mykey.pub
└── sv
    └── xvfb
        ├── log
        │   └── run
        └── run
```

### Dockerfile

最初に完成したDockerfileです。

``` bash ~/docker_apps/eclim/Dockerfile
FROM phusion/baseimage:0.9.15
MAINTAINER Masato Shimizu <ma6ato@gmail.com>
# Set correct environment variables.
ENV HOME /root
# Regenerate SSH host keys.
RUN /etc/my_init.d/00_regen_ssh_host_keys.sh
## Install an SSH of your choice.
ADD mykey.pub /tmp/your_key
RUN cat /tmp/your_key >> /root/.ssh/authorized_keys \
  && chmod 600 /root/.ssh/authorized_keys \
  && chmod 600 /root/.ssh/authorized_keys && rm -f /tmp/your_key
## apt-get update
RUN sed -i~ -e 's/archive.ubuntu.com/ftp.jaist.ac.jp/' /etc/apt/sources.list \
 && apt-get update
## Japanese Environment
RUN DEBIAN_FRONTEND=noninteractive apt-get install -y language-pack-ja
ENV LANG ja_JP.UTF-8
RUN update-locale LANG=ja_JP.UTF-8
RUN mv /etc/localtime /etc/localtime.org
RUN ln -s /usr/share/zoneinfo/Asia/Tokyo /etc/localtime
RUN DEBIAN_FRONTEND=noninteractive \
    apt-get install -y build-essential software-properties-common \
                       zlib1g-dev libssl-dev libreadline-dev libyaml-dev \
                       libxml2-dev libxslt-dev sqlite3 libsqlite3-dev \
                       emacs24-nox emacs24-el git byobu wget curl unzip tree\
                       python
# non-root user
RUN useradd -m -d /home/docker -s /bin/bash docker \
 && echo "docker:docker" | chpasswd \
 && mkdir /home/docker/.ssh \
 && chmod 700 /home/docker/.ssh \
 && cp /root/.ssh/authorized_keys /home/docker/.ssh \
 && chmod 600 /home/docker/.ssh/authorized_keys \
 && echo "export LANG=ja_JP.UTF-8" >> /home/docker/.profile \
 && chown -R docker:docker /home/docker/.ssh
RUN echo "docker ALL=(ALL) NOPASSWD:ALL" >> /etc/sudoers
# eclim
RUN apt-get install -y openjdk-7-jdk ant maven \
    xvfb xfonts-100dpi xfonts-75dpi xfonts-scalable xfonts-cyrillic
ADD .emacs.d /home/docker/.emacs.d
RUN chown -R docker:docker /home/docker/.emacs.d
USER docker
ENV HOME /home/docker
WORKDIR /home/docker
# Cask
RUN curl -fsSkL https://raw.github.com/cask/cask/master/go | python
RUN /bin/bash -c 'echo export PATH="/home/docker/.cask/bin:$PATH" >> /home/docker/.profile' \
 && /bin/bash -c 'source /home/docker/.profile && cd /home/docker/.emacs.d && cask install'
# eclipse,eclim
RUN wget -P /home/docker http://ftp.yz.yamagata-u.ac.jp/pub/eclipse/technology/epp/downloads/release/luna/R/eclipse-java-luna-R-linux-gtk-x86_64.tar.gz \
 && tar xzvf eclipse-java-luna-R-linux-gtk-x86_64.tar.gz -C /home/docker \
 && mkdir /home/docker/workspace \
 && cd /home/docker && git clone git://github.com/ervandew/eclim.git \
 && cd eclim && ant -Declipse.home=/home/docker/eclipse
USER root
ADD sv /etc/service
CMD ["/sbin/my_init"]
RUN apt-get clean && rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*
```

### Xvfbのrunit

runitの起動スクリプトを作成します。XvfbはIPv6を無効にするため`-nolisten inet6`のフラグをつけます。

``` bash ~/docker_apps/eclim/sv/xvfb/run
#!/bin/sh
exec 2>&1
exec Xvfb :1 -screen 0 1024x768x24 -nolisten inet6
```

ログの設定です。

``` bash ~/docker_apps/eclim/sv/xvfb/log/run
!/bin/sh
service=$(basename $(dirname $(pwd)))
logdir="/var/log/${service}"
mkdir -p ${logdir}
exec 2>&1
exec /usr/bin/svlogd -tt ${logdir}
```

それぞれ実行権限をつけます。

``` bash
$ chmod +x sv/xvfb/run
$ chmod +x sv/xvfb/log/run
```

### .emacs.d

[Cask](http://cask.github.io/)の設定ファイルです。auto-completeとemacs-eclimをインストールします。

``` el ~/docker_apps/eclim/.emacs.d/Cask
(source gnu)
(source marmalade)
(source melpa)
(depends-on "pallet")
(depends-on "init-loader")
;; emacs-eclim
(depends-on "auto-complete")
(depends-on "emacs-eclim")
```

[Symbol's function definition is void: find](https://github.com/senny/emacs-eclim/issues/95)のissuesを読むとemacs 24.3の場合には、`(require 'cl)`が必要です。

``` el ~/docker_apps/eclim/.emacs.d/inits/04-eclim.el
(require 'cl)
(require 'eclim)
(global-eclim-mode)
(custom-set-variables
  '(eclim-eclipse-dirs '("/home/docker/eclipse"))
  '(eclim-executable "/home/docker/eclipse/eclim")
  '(eclimd-default-workspace "/home/docker/workspace"))
;; regular auto-complete initialization
(require 'auto-complete-config)
(ac-config-default)
;; add the emacs-eclim source
(require 'ac-emacs-eclim-source)
(ac-emacs-eclim-config)
(ac-set-trigger-key "TAB")
(define-key ac-complete-mode-map (kbd "C-n") 'ac-next)
(define-key ac-complete-mode-map (kbd "C-p") 'ac-previous)
(add-hook 'java-mode-hook 'eclim-mode)
```

以下はいつもと同じです。init.elで`init-loader`を使えるようにします。

``` el ~/docker_apps/eclim/.emacs.d/init.el
(require 'cask "~/.cask/cask.el")
(cask-initialize)
(require 'pallet)

(require 'init-loader)
(setq init-loader-show-log-after-init nil)
(init-loader-load "~/.emacs.d/inits")
```

キーバインドの設定です。

``` el ~/docker_apps/eclim/.emacs.d/inits/00-keybinds.el
(define-key global-map "\C-h" 'delete-backward-char)
(define-key global-map "\M-?" 'help-for-help)
```

ファイルの設定です。

``` el el ~/docker_apps/eclim/.emacs.d/inits/01-files.el
(setq backup-inhibited t)
(setq next-line-add-newlines nil)
(setq-default tab-width 4 indent-tabs-mode nil)
```

### コンテナを起動して確認する

この後トラブルシュートと個別のファイルについて書きますが、まずは動作確認をします。
コンテナを起動してSSH接続します。

``` bash
$ cd ~/docker_apps/eclim
$ docker build -t masato/eclim .
$ docker run --name eclim -d -i -t masato/eclim
$ eval `ssh-agent`
$ ssh-add ~/.ssh/mykey
$ ssh docker@$(docker inspect --format="&#123;&#123; .NetworkSettings.IPAddress }}" eclim)
```
eclimd を起動します。

``` bash
$ DISPLAY=:1 ./eclipse/eclimd -b
```

emacsを起動します。

``` bash
$ emacs
```

プロジェクトを作成します。名前はexampleとしてjavaのプロジェクトを作成します。

``` bash
M-x eclim-project-create
Name: example
Project Directory: ~/workspace/example
Type: java
Created project 'example'.
```

作成したプロジェクトを確認します。

``` bash
M-x eclim-project-mode

  | open   | example                        | /home/docker/workspace/example
```


### Eclipse 4.4.0を使う

以下はトラブルシュートのメモです。

`phusion/baseimage:0.9.15`はUbuntu 14.04を使っています。

``` bash /etc/lsb-release
DISTRIB_ID=Ubuntu
DISTRIB_RELEASE=14.04
DISTRIB_CODENAME=trusty
DISTRIB_DESCRIPTION="Ubuntu 14.04.1 LTS"
```

apt-getでインストールできるEclipseのバージョン古いです。

``` bash
$ apt-cache show eclipse-platform | grep Version
Version: 3.8.1-5.1
```

[Eclipse Luna (4.4.1 SR1)](http://stackoverflow.com/questions/26129731/exception-starting-eclipse-luna-sr1-on-osx)はeclimdの実行時に例外が発生します。

``` bash
$ DISPLAY=:1 ./eclipse/eclimd -b
...
java.lang.ClassCastException: org.eclipse.osgi.internal.framework.EquinoxConfiguration$1 cannot be cast to java.lang.String
        at org.eclipse.m2e.logback.configuration.LogHelper.logJavaProperties(LogHelper.java:26)
        at org.eclipse.m2e.logback.configuration.LogPlugin.loadConfiguration(LogPlugin.java:189)
        at org.eclipse.m2e.logback.configuration.LogPlugin.configureLogback(LogPlugin.java:144)
        at org.eclipse.m2e.logback.configuration.LogPlugin.access$2(LogPlugin.java:107)
        at org.eclipse.m2e.logback.configuration.LogPlugin$1.run(LogPlugin.java:62)
        at java.util.TimerThread.mainLoop(Timer.java:555)
        at java.util.TimerThread.run(Timer.java:505)
```

Eclipse Luna (4.4.0)をダウンロードして使います。

``` bash ~/docker_apps/eclim/Dockerfile
RUN wget -P /home/docker http://ftp.yz.yamagata-u.ac.jp/pub/eclipse/technology/epp/downloads/release/luna/R/eclip\
se-java-luna-R-linux-gtk-x86_64.tar.gz \
```

### eclimはgit cloneしてビルドする

配布されている`eclim_2.4.0.jar`には、[NullPointer when attempting to install eclim headless]( https://groups.google.com/forum/#!topic/eclim-user/9H_C8bYJJ-E)や[Can not install eclim](http://stackoverflow.com/questions/25580805/can-not-install-eclim)にissuesがあるように、NPEが発生するため`git clone`して最新バージョンをビルドします。

``` bash
$ java \
  -Declipse.home=/opt/eclipse \
  -Dvim.skip=true \
  -jar eclim_2.4.0.jar install
...
2014-10-03 08:49:47,738 DEBUG [ANT]
BUILD SUCCESSFUL
Total time: 19 seconds
java.lang.NullPointerException
```

また、antを使ったeclimのビルドは、rootで行うとエラーになります。

``` bash
Buildfile: /root/eclim/build.xml

deploy:

gant:

BUILD FAILED
/root/eclim/build.xml:32: The following error occurred while executing this line:
: Abort:

    #####
    Building eclim as root is highly discouraged.
    #####

Total time: 3 seconds
```

そのためnon-rootのユーザーにスイッチしてビルドを行います。

``` bash ~/docker_apps/eclim/Dockerfile
...
 && cd /home/docker && git clone git://github.com/ervandew/eclim.git \
 && cd eclim && ant -Declipse.home=/home/docker/eclipse
```

### Xvfbはipv6を無効にする

Dockerホストでipv6を無効にしているため、Xvfbの起動でエラーが発生します。

``` bash
$ Xvfb :1 -screen 0 1024x768x24 &
[1] 28
docker@001ab1d4a533:~$ _XSERVTransSocketOpenCOTSServer: Unable to open socket for inet6
_XSERVTransOpen: transport open failed for inet6/001ab1d4a533:1
_XSERVTransMakeAllCOTSServerListeners: failed to open listener for inet6
_XSERVTransmkdir: ERROR: euid != 0,directory /tmp/.X11-unix will not be created.
```

また、rootで起動していないと警告が発生します。

``` bash
$ Xvfb :1 -screen 0 1024x768x24 -nolisten inet6 &
[1] 220
docker@2463c62b0ee3:~$ _XSERVTransmkdir: Owner of /tmp/.X11-unix should be set to root
```

従ってsudoでipv6を無効にして起動します。

``` bash
$ sudo Xvfb :1 -screen 0 1024x768x24 -nolisten inet6 &
[1] 251
docker@2463c62b0ee3:~$ Initializing built-in extension Generic Event Extension
Initializing built-in extension SHAPE
Initializing built-in extension MIT-SHM
Initializing built-in extension XInputExtension
Initializing built-in extension XTEST
Initializing built-in extension BIG-REQUESTS
Initializing built-in extension SYNC
Initializing built-in extension XKEYBOARD
Initializing built-in extension XC-MISC
Initializing built-in extension SECURITY
Initializing built-in extension XINERAMA
Initializing built-in extension XFIXES
Initializing built-in extension RENDER
Initializing built-in extension RANDR
Initializing built-in extension COMPOSITE
Initializing built-in extension DAMAGE
Initializing built-in extension MIT-SCREEN-SAVER
Initializing built-in extension DOUBLE-BUFFER
Initializing built-in extension RECORD
Initializing built-in extension DPMS
Initializing built-in extension Present
Initializing built-in extension DRI3
Initializing built-in extension X-Resource
Initializing built-in extension XVideo
Initializing built-in extension XVideo-MotionCompensation
Initializing built-in extension SELinux
Initializing built-in extension GLX
```
