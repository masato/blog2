title: "Docker開発環境をつくる - Emacs24とEclimでOracle Java 8のSpring Boot開発環境は失敗したのでOpenJDK 7を使う"
date: 2014-10-06 02:24:50
tags:
 - DockerDevEnv
 - Java
 - Emacs
 - Cask
 - Eclim
description: Eclimを使いSpringBootを試すことができたので、もう少し開発環境を整えていきます。Webアプリの開発用にweb-modeなどEmacsパッケージを追加します。expand-region.elをはじめて使ってみました。最近の中で一番便利な気がします。大きな範囲のリージョン選択でキータイプが減ってうれしいです。これからはOracle Java 8に切り替えてモダンなJavaも勉強していきたいのですが、eclimdの起動に失敗してしまいました。仕方がないのでopenjdk-7-jdkのまま使うことにします。
---

Eclimを使い[SpringBootの開発](/2014/10/05/spring-boot-emacs-eclim-helloworld/)を試すことができたので、もう少し開発環境を整えていきます。
Webアプリの開発用に[web-mode](http://web-mode.org/)などEmacsパッケージを追加します。[expand-region.el](https://github.com/magnars/expand-region.el)をはじめて使ってみました。最近の中で一番便利な気がします。大きな範囲のリージョン選択でキータイプが減ってうれしいです。
これからは`Oracle Java 8`に切り替えてモダンなJavaも勉強していきたいのですが、eclimdの起動に失敗してしまいました。仕方がないのでopenjdk-7-jdkのまま使うことにします。

<!-- more -->


### Dockerfile

`Oracler Java 8`用に編集したDockerfileです。

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
# Add Un-Root User
RUN useradd -m -d /home/docker -s /bin/bash docker \
 && echo "docker:docker" | chpasswd \
 && mkdir /home/docker/.ssh \
 && chmod 700 /home/docker/.ssh \
 && cp /root/.ssh/authorized_keys /home/docker/.ssh \
 && chmod 600 /home/docker/.ssh/authorized_keys \
 && echo "export LANG=ja_JP.UTF-8" >> /home/docker/.profile \
 && chown -R docker:docker /home/docker/.ssh
RUN echo "docker ALL=(ALL) NOPASSWD:ALL" >> /etc/sudoers
# oracle-java8 didn't work well with eclim
#RUN echo oracle-java8-installer shared/accepted-oracle-license-v1-1 select true | debconf-set-selections \
# && add-apt-repository -y ppa:webupd8team/java \
# && apt-get update \
# && apt-get install -y oracle-java8-installer oracle-java8-set-default
# openjdk-7-jdk,eclim
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
# Spring Boot
RUN add-apt-repository ppa:cwchien/gradle \
 && apt-get update \
 && apt-get install -y gradle-ppa
ADD sv /etc/service
CMD ["/sbin/my_init"]
RUN apt-get clean && rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*
```

### Oracle Java JDK

通常Oracle Javaをインストールする際はラインセンスに承諾するためダイアログが表示されます。Dockerでサイレントインストールするために事前設定ファイルを作成します。


``` bash ~/docker_apps/eclim/Dockerfile
# oracle-java8
RUN echo oracle-java8-installer shared/accepted-oracle-license-v1-1 select true | debconf-set-selections \
 && add-apt-repository -y ppa:webupd8team/java \
 && apt-get update \
 && apt-get install -y oracle-java8-installer oracle-java8-set-default
```
`oracle-java8-set-default`をインストールすると環境設定ファイルが作成されます。

``` bash /etc/profile.d/jdk.sh
export J2SDKDIR=/usr/lib/jvm/java-8-oracle
export J2REDIR=/usr/lib/jvm/java-8-oracle/jre
export PATH=$PATH:/usr/lib/jvm/java-8-oracle/bin:/usr/lib/jvm/java-8-oracle/db/bin:/usr/lib/jvm/java-8-oracle/jre/bin
export JAVA_HOME=/usr/lib/jvm/java-8-oracle
export DERBY_HOME=/usr/lib/jvm/java-8-oracle/db
```

nsenterやSSHでコンテナにログインした時に`JAVA_HOME`が設定されています。

``` bash
$ ssh docker@172.17.0.238
$ echo $JAVA_HOME
/usr/lib/jvm/java-8-oracle
```

### Caskファイル

[Cask](http://cask.github.io/)ファイルです。multi-termやexpand-regionのパッケージを追加します。

``` el ~/docker_apps/eclim/.emacs.d/Cask
(source gnu)
(source marmalade)
(source melpa)
(depends-on "pallet")
(depends-on "init-loader")
(depends-on "multi-term")
(depends-on "expand-region")
(depends-on "color-theme-solarized")
;; web-mode
(depends-on "web-mode")
;; emacs-eclim
(depends-on "auto-complete")
(depends-on "emacs-eclim")
```

### init-loader.el

init-loader.elのinitsにいくつか初期設定ファイルを追加します。

``` bash
$ tree -a ~/docker_apps/eclim/
~/docker_apps/eclim/
├── .emacs.d
│   ├── Cask
│   ├── init.el
│   └── inits
│       ├── 00-keybinds.el
│       ├── 01-files.el
│       ├── 03-multi-term.el
│       ├── 04-eclim.el
│       ├── 05-color-theme-solarized.el
│       ├── 06-web-mode.el
│       └── 07-expand-region.el
├── Dockerfile
├── mykey.pub
└── sv
    └── xvfb
        ├── log
        │   └── run
        └── run
```

[multi-term](http://www.emacswiki.org/MultiTerm)の設定です。

``` el  ~/docker_apps/eclim/.emacs.d/inits/03-multi-term.el
(require 'multi-term)
(setq multi-term-program shell-file-name)
```

[Solarized Colorscheme for Emacs](https://github.com/sellout/emacs-color-theme-solarized)のテーマを追加します。

``` el  ~/docker_apps/eclim/.emacs.d/inits/color-theme-solarized.el
(load-theme 'solarized-dark t)
```

iTermを使っているとDiredのディレクトリが青く見づらいので、iTremのカラーを変更する。

```
Preferences -> Colors -> Load Presets -> Tango Dark
```


HTMLファイルの編集用に[web-mode](http://web-mode.org/)を追加します。faceで色がつかないので、`custom-set-faces`で以下のページを参考に色の設定をします。

* [emacsでviewファイルをいじるならweb-modeを使うべき](http://wadap.hatenablog.com/entry/2013/09/10/101315)

``` el  ~/docker_apps/eclim/.emacs.d/inits/06-web-mode.el
(require 'web-mode)
(add-to-list 'auto-mode-alist '("\\.[gj]sp\\'" . web-mode))
(add-to-list 'auto-mode-alist '("\\.html?\\'" . web-mode))

(add-hook
 'web-mode-hook
  (lambda ()
   (setq web-mode-markup-indent-offset 2)
   (setq web-mode-css-indent-offset 2)
   (setq web-mode-code-indent-offset 2)))

(custom-set-faces
 '(web-mode-doctype-face
   ((t (:foreground "#82AE46"))))
 '(web-mode-html-tag-face
   ((t (:foreground "#E6B422" :weight bold))))
 '(web-mode-html-attr-name-face
   ((t (:foreground "#C97586"))))
 '(web-mode-html-attr-value-face
   ((t (:foreground "#82AE46"))))
 '(web-mode-comment-face
   ((t (:foreground "#D9333F"))))
 '(web-mode-server-comment-face
   ((t (:foreground "#D9333F"))))
 '(web-mode-css-rule-face
   ((t (:foreground "#A0D8EF"))))
 '(web-mode-css-pseudo-class-face
   ((t (:foreground "#FF7F00"))))
 '(web-mode-css-at-rule-face
   ((t (:foreground "#FF7F00"))))
)
```

[expand-region.el](https://github.com/magnars/expand-region.el)の設定は以下を参考にしました。

* [Emacsで選択範囲をインタラクティブに広げる expand-region](http://qiita.com/ongaeshi/items/abd1016bf484c4e05ab1)

``` el  ~/docker_apps/eclim/.emacs.d/inits/07-expand-region.el
(require 'expand-region)
(global-set-key (kbd "C-@") 'er/expand-region)
(global-set-key (kbd "C-M-@") 'er/contract-region)
```

### Oracke Javaだとeclimdが起動しない

`OpenJDK 7`ではeclimdが起動するのですが、`Oracke Java 8`と`Oracke Java 7`を使うとeclimdの起動に失敗します。
まだ原因がわからないのでデバッグ中です。

``` bash
$ DISPLAY=:1 ./eclipse/eclimd -b
Java HotSpot(TM) 64-Bit Server VM warning: ignoring option MaxPermSize=256m; support was removed in 8.0

docker@639d66636d75:~$ Exception in thread "Thread-4" java.lang.NoClassDefFoundError: org/eclipse/ui/PlatformUI
        at org.eclim.eclipse.EclimApplication.shutdown(EclimApplication.java:135)
        at org.eclim.eclipse.EclimApplication$1.run(EclimApplication.java:93)
Caused by: java.lang.ClassNotFoundException: org.eclipse.ui.PlatformUI cannot be found by org.eclim_2.4.0.25-g65a1584
        at org.eclipse.osgi.internal.loader.BundleLoader.findClassInternal(BundleLoader.java:423)
        at org.eclipse.osgi.internal.loader.BundleLoader.findClass(BundleLoader.java:336)
        at org.eclipse.osgi.internal.loader.BundleLoader.findClass(BundleLoader.java:328)
        at org.eclipse.osgi.internal.loader.ModuleClassLoader.loadClass(ModuleClassLoader.java:160)
        at java.lang.ClassLoader.loadClass(ClassLoader.java:357)
        ... 2 more
```

### OpenJDK 7だとeclimdが起動できる

Dockerfileを修正してOpenJDK 7をインストールします。以前の通りにeclimdが起動しました。
[Utopicからopenjdk-8-jdk](http://packages.ubuntu.com/utopic/openjdk-8-jdk)がパッケージからインストールできるのでリリースまでしばらく待とうと思います。

``` bash
$ DISPLAY=:1 ./eclipse/eclimd -b
$ ps ux | grep "[e]clim"
docker     122  3.7  5.8 3199800 480900 pts/0  Sl   19:58   0:37 /usr/bin/java -d64 -Dosgi.requiredJavaVersion=1.6 -XX:MaxPermSize=256m -Xms40m -Xmx512m -jar /home/docker/eclipse/plugins/org.eclipse.equinox.launcher_1.3.0.v20140415-2008.jar -debug -clean -refresh -application org.eclim.application
```






