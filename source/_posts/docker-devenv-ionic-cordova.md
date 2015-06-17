title: "Dockerで開発環境をつくる - IonicとCordovaとAndroid SDKをインストール"
date: 2014-12-30 00:20:10
tags:
 - MBaaS
 - DockerDevEnv
 - Ionic
 - Cordova
 - OnsenUI
 - AndroidSDK
 - HTML5HybridMobileApps
description: MBaaSを調査しているところですがCordovaをベースにしてIonicやOnsen UI、SupersonicをつかったHTML5ハイブリッドモバイルアプリを作ってみようと思います。開発環境はなるべくクラウド上のDockerコンテナで行いたいです。DockerコンテナにAndroid SDKはヘッドレスでインストールしてIonicの開発環境を構築します。ローカルのサーバーを起動することでリモートからブラウザでプレビューができるようになります。
---

[MBaaSを調査](/2014/12/22/mbaas-html5-hybrid-mobile-apps-references/)しているところですが[Cordova](http://cordova.apache.org/)をベースにして[Ionic](http://ionicframework.com/)や[Onsen UI](http://onsen.io/)、[Supersonic](http://www.appgyver.com/supersonic)をつかったHTML5ハイブリッドモバイルアプリを作ってみようと思います。開発環境はなるべくクラウド上のDockerコンテナで行いたいです。DockerコンテナにAndroid SDKはヘッドレスでインストールしてIonicの開発環境を構築します。ローカルのサーバーを起動することでリモートからブラウザでプレビューができるようになります。

<!-- more -->

### Dockerfile

今回作成したDockerイメージは、[masato/ionic-cordova](https://registry.hub.docker.com/u/masato/ionic-cordova/)にpushしています。GitHubのリポジトリは[ここ](https://github.com/masato/ionic-cordova)です。開発環境用のDockerイメージは[phusion/baseimage](https://index.docker.io/u/phusion/baseimage/)をベースにしています。


``` bash ~/docker_apps/ionic-cordova/Dockerfile
FROM phusion/baseimage:0.9.15
MAINTAINER Masato Shimizu <ma6ato@gmail.com>
# Set correct environment variables.
ENV HOME /root
# Regenerate SSH host keys.
RUN /etc/my_init.d/00_regen_ssh_host_keys.sh
## Enabling the insecure key permanently
RUN /usr/sbin/enable_insecure_key
## apt-get update
## oracle-java8
RUN add-apt-repository ppa:webupd8team/java && \
    echo oracle-java7-installer shared/accepted-oracle-license-v1-1 select true | /usr/bin/debconf-set-selections
RUN apt-get update && \
    DEBIAN_FRONTEND=noninteractive \
    apt-get install -y build-essential software-properties-common \
                       python emacs24-nox emacs24-el \
                       git byobu wget curl unzip tree elinks \
                       oracle-java7-installer oracle-java7-set-default ant \
                       lib32z1 lib32ncurses5 lib32bz2-1.0 lib32stdc++6
ENV JAVA_HOME /usr/lib/jvm/java-7-oracle
## Japanese Environment
ENV LANG ja_JP.UTF-8
RUN locale-gen $LANG && update-locale $LANG && \
    ln -sf /usr/share/zoneinfo/Asia/Tokyo /etc/localtime
## Development Environment
RUN update-alternatives --set editor /usr/bin/vim.basic
ENV USERNAME docker
ENV HOME /home/${USERNAME}
ENV PATH $PATH:${HOME}/bin
# Add Un-Root User
RUN useradd -m -d ${HOME} -s /bin/bash ${USERNAME} && \
    echo "${USERNAME}:${USERNAME}" | chpasswd && \
    mkdir ${HOME}/.ssh ${HOME}/tmp && \
    chmod 700 ${HOME}/.ssh && \
    cp /root/.ssh/authorized_keys ${HOME}/.ssh && \
    chmod 600 ${HOME}/.ssh/authorized_keys && \
    chown -R ${USERNAME}:${USERNAME} ${HOME}/.ssh && \
    echo "export LANG=ja_JP.UTF-8" >> ${HOME}/.profile && \
    echo "docker ALL=(ALL) NOPASSWD:ALL" >> /etc/sudoers
ENV USERNAME docker
ENV HOME /home/${USERNAME}
WORKDIR /data
ENV DATA_DIR /data
ADD dotfiles ${HOME}
RUN mkdir -p /data && \
    chown -R ${USERNAME}:${USERNAME} ${HOME} ${DATA_DIR}
USER ${USERNAME}
## Cask
RUN curl -fsSkL https://raw.github.com/cask/cask/master/go | python && \
    echo export PATH='${HOME}/.cask/bin:$PATH' >> ${HOME}/.profile && \
    /bin/bash -c 'source ${HOME}/.profile && cd ${HOME}/.emacs.d && cask install'
## auto-java-complete
RUN cd ${HOME}/.emacs.d && \
    git clone https://github.com/emacs-java/auto-java-complete && \
    cd auto-java-complete && \
    javac Tags.java -encoding UTF-8 && \
    java -cp "/usr/lib/jvm/java-7-oracle/jre/lib/rt.jar:." Tags
## Android SDK
ENV ANDROID_SDK_VERSION r24.0.2
ENV ANDROID_API_LEVEL 19
ENV ANDROID_BUILD_TOOLS_VERSION 21.1.1
ENV ANDROID_HOME ${HOME}/android-sdk-linux
RUN echo 'export ANDROID_HOME=${ANDROID_HOME}' >> ${HOME}/.profile
ENV PATH ${PATH}:${ANDROID_HOME}/tools:${ANDROID_HOME}/platform-tools
RUN echo 'export PATH=${PATH}:${ANDROID_HOME}/tools:${ANDROID_HOME}/platform-tools' >> ${HOME}/.profile
RUN wget -qO- "http://dl.google.com/android/android-sdk_${ANDROID_SDK_VERSION}-linux.tgz" \
    | tar -zxv -C ${HOME} && \
    echo y | android update sdk --no-ui --all --filter platform-tool,android-${ANDROID_API_LEVEL},sysimg-${ANDROID_API_LEVEL},build-tools-${ANDROID_BUILD_TOOLS_VERSION}
ENV NODE_VERSION v0.10
## Node.js/Cordova/Ionic
RUN curl https://raw.githubusercontent.com/creationix/nvm/v0.22.0/install.sh | bash && \
    /bin/bash -c 'source ${HOME}/.nvm/nvm.sh && \
                  nvm install ${NODE_VERSION} && \
                  nvm use ${NODE_VERSION} && \
                  nvm alias default ${NODE_VERSION} && \
                  npm install -g npm && \
                  npm install -g cordova ionic && \
                  npm cache clear'
## Create Cordova and ionic sample apps
RUN /bin/bash -c 'source ${HOME}/.nvm/nvm.sh && nvm use ${NODE_VERSION} \
    cd ${DATA_DIR} && \
    cordova create cordova_test com.example.test "CordovaTestApp" && \
    cd ${DATA_DIR}/cordova_test && \
    cordova platform add android && \
    cordova build && \
    cd ${DATA_DIR} && \
    ionic start ionicTestApp tabs && \
    cd ${DATA_DIR}/ionicTestApp && \
    ionic platform add android && \
    ionic build android'
USER root
ENV HOME /root
VOLUME ["/data"]
EXPOSE 8100 35729
CMD ["/sbin/my_init"]
RUN apt-get clean && rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*
```

### 32bit版のライブラリ

IonicのインストールにはOracle JDK 7、Ant、Android SDK、Node.jsが必要です。今回のDockerコンテナはUbuntu 14.04の64bit版ですが、Androidが一部32bit版のライブラリに依存しているようです。[Installing and setting up Android Developer Tools in Ubuntu 13.10](http://sherwinrobles.blogspot.jp/2014/02/installing-and-setting-up-android.html)を参考にして、`lib32z1 lib32ncurses5 lib32bz2-1.0 lib32stdc++6`のライブラリをインストールします。

### Android SDK

Android SDKはヘッドレスで使います。今回の設定は以下のフィルタしました。

> android update sdk --no-ui --all --filter platform-tool,android-19,sysimg-19,build-tools-21.1.1

APIレベルは[uses-sdk](http://developer.android.com/guide/topics/manifest/uses-sdk-element.html)を参照にしてandroid-19を指定しました。Android 4.4(Kitkat)を対象にします。

Build Toolsのバージョンも [SDK Build Tools Release Notes](https://developer.android.com/tools/revisions/build-tools.html)を参照して、最新の21.1.1を指定しています。

### CordovaとIonic

CordoavとIonicはどちらもnpmでインストールするためNode.jsが必要です。nvmでv0.10.35をインストールします。そのあと確認のためCordovaとIonicのサンプルを`/data`ディレクトリに作成しました。

### Dockerコンテナの起動

使い捨てのDockerコンテナを起動します。開発ユーザーの`docker`にスイッチします。

``` bash
$ docker run -it --rm masato/ionic-cordova /sbin/my_init bash
$ su - docker
```

Ionicのサンプルアプリのディレクトリに移動します。nvmを使い`ionic serve`を実行してローカルにWebサーバーを起動します。

``` bash
$ cd /data/ionicTestApp/
$ ionic serve

Multiple addresses available.
Please select which address to use by entering its number from the list below:
 1) 172.17.3.90 (eth0)
 2) localhost
Address Selection:  1
Selected address: 172.17.3.90
Running dev server: http://172.17.3.90:8100
Running live reload server: http://172.17.3.90:35729
Watching : [ 'www/**/*', '!www/lib/**/*' ]
Ionic server commands, enter:
  restart or r to restart the client app from the root
  goto or g and a url to have the app navigate to the given url
  consolelogs or c to enable/disable console log output
  serverlogs or s to enable/disable server log output
  quit or q to shutdown the server and exit
```

DockerコンテナのIPアドレスを確認して、ngrokでトンネルします。

``` bash
$ docker inspect --format="&#123;&#123; .NetworkSettings.IPAddress }}" e0c6dac55a79
172.17.3.90
$ docker run -it --rm wizardapps/ngrok:latest ngrok 172.17.3.90:8100
```

ブラウザでngrokのトンネルしたURLを開くと、Ionicのtabsサンプルをプレビューできます。

http://6e7e14f6.ngrok.com/

{% img center /2014/12/30/docker-devenv-ionic-cordova/ionic-chats.png %}



