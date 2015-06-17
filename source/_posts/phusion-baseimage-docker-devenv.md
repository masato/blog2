title: "Phusion Baseimage-dockerを使ったDocker開発環境"
date: 2014-11-28 21:08:13
tags:
 - baseimage
 - DockerDevEnv
 - Cask
 - Emacs
 - Eclim
---


[phusion/baseimage-docker](phusion/baseimage-docker)を使ったDocker開発環境の[baseimage](https://registry.hub.docker.com/u/masato/baseimage/)を作りました。Docker Hub Registryに公開しています。Caskでパッケージをインスト-ルしたEmacs/Eclimをエディタに使い、Scala、Dart、Go、Ruby、Python、Node.js、Clojureの開発ができます。

<!-- more -->

### Dockerfile

GitHubのリポジトリは[masato/baseimage](https://github.com/masato/baseimage)です。
秘密鍵はphusionの[insecure_key](https://github.com/phusion/baseimage-docker)を使っているので、プロダクション環境では使わないでください。

``` bash
$ curl -o insecure_key -fSL https://github.com/phusion/baseimage-docker/raw/master/image/insecure_key
$ chmod 600 insecure_key
```

Dockerfileです。

``` bash
FROM phusion/baseimage:0.9.15
MAINTAINER Masato Shimizu <ma6ato@gmail.com>

ENV RUBY_VERSION 2.1.5
ENV TYPESAFE_VERSION 1.2.10

# Set correct environment variables.
ENV HOME /root

# Regenerate SSH host keys.
RUN /etc/my_init.d/00_regen_ssh_host_keys.sh

## Enabling the insecure key permanently
RUN /usr/sbin/enable_insecure_key

## apt-get update
RUN apt-get update && \
    DEBIAN_FRONTEND=noninteractive \
    apt-get install -y build-essential software-properties-common \
                       zlib1g-dev libssl-dev libreadline-dev libyaml-dev \
                       libxml2-dev libxslt-dev sqlite3 libsqlite3-dev \
                       emacs24-nox emacs24-el git byobu wget curl unzip tree \
                       python-dev python-pip golang nodejs npm \
                       openjdk-7-jdk ant maven \
                       xvfb xfonts-100dpi xfonts-75dpi xfonts-scalable xfonts-cyrillic

## Japanese Environment
ENV LANG ja_JP.UTF-8
RUN locale-gen $LANG && update-locale $LANG && \
    ln -sf /usr/share/zoneinfo/Asia/Tokyo /etc/localtime

## Development Environment
RUN update-alternatives --set editor /usr/bin/vim.basic

# Add Un-Root User
RUN useradd -m -d /home/docker -s /bin/bash docker && \ 
    echo "docker:docker" | chpasswd && \
    mkdir /home/docker/.ssh /home/docker/tmp && \
    chmod 700 /home/docker/.ssh && \
    cp /root/.ssh/authorized_keys /home/docker/.ssh && \
    chmod 600 /home/docker/.ssh/authorized_keys && \
    chown -R docker:docker /home/docker/.ssh && \
    echo "export LANG=ja_JP.UTF-8" >> /home/docker/.profile && \
    echo "docker ALL=(ALL) NOPASSWD:ALL" >> /etc/sudoers

## Ruby rbenv install
RUN git clone https://github.com/sstephenson/ruby-build.git .ruby-build && \
    .ruby-build/install.sh && \
    rm -fr .ruby-build && \
    ruby-build $RUBY_VERSION /usr/local && \
    gem update --system && \
    gem install bundler --no-rdoc --no-ri

## Node.js
RUN ln -s /usr/bin/nodejs /usr/bin/node

ENV USERNAME docker
ENV HOME /home/${USERNAME}
ENV PATH $PATH:${HOME}/bin

ADD dotfiles ${HOME}
RUN chown -R ${USERNAME}:${USERNAME} ${HOME}

USER ${USERNAME}
WORKDIR ${HOME}

## Go
RUN mkdir -p ${HOME}/gocode/src ${HOME}/gocode/bin ${HOME}/gocode/pkg && \
    echo 'export GOPATH=${HOME}/gocode' >> ${HOME}/.profile && \
    echo 'export PATH=${PATH}:${HOME}/gocode/bin' >> ${HOME}/.profile


## Dart
RUN wget http://storage.googleapis.com/dart-archive/channels/stable/release/latest/sdk/dartsdk-linux-x64-release.zip -P ${HOME} && \
    unzip ${HOME}/dartsdk-linux-x64-release.zip -d ${HOME} && \
    echo 'export PATH=${PATH}:${HOME}/dart-sdk/bin' >> ${HOME}/.profile && \
    rm ${HOME}/dartsdk-linux-x64-release.zip

## Scala
RUN mkdir -p ${HOME}/bin && \
   wget http://downloads.typesafe.com/typesafe-activator/${TYPESAFE_VERSION}/typesafe-activator-${TYPESAFE_VERSION}.zip -P ${HOME} && \
   unzip ${HOME}/typesafe-activator-${TYPESAFE_VERSION}.zip -d ${HOME} && \
   ln -s ${HOME}/activator-${TYPESAFE_VERSION}/activator ${HOME}/bin/activator && \
   rm ${HOME}/typesafe-activator-${TYPESAFE_VERSION}.zip

# Cask
RUN curl -fsSkL https://raw.github.com/cask/cask/master/go | python && \
    /bin/bash -c 'echo export PATH="${HOME}/.cask/bin:$PATH" >> ${HOME}/.profile' && \
    /bin/bash -c 'source ${HOME}/.profile && cd ${HOME}/.emacs.d && cask install'

# eclipse,eclim
RUN wget -P ${HOME} http://ftp.yz.yamagata-u.ac.jp/pub/eclipse/technology/epp/downloads/release/luna/R/eclipse-java-luna-R-linux-gtk-x86_64.tar.gz && \
    tar xzvf ${HOME}/eclipse-java-luna-R-linux-gtk-x86_64.tar.gz -C ${HOME} && \
    mkdir ${HOME}/workspace && \
    cd ${HOME} && git clone git://github.com/ervandew/eclim.git && \
    cd eclim && ant -Declipse.home=${HOME}/eclipse && \
    rm ${HOME}/eclipse-java-luna-R-linux-gtk-x86_64.tar.gz 

## Clojure
RUN curl -s https://raw.githubusercontent.com/technomancy/leiningen/2.5.0/bin/lein > \
    ${HOME}/bin/lein && \
    chmod 0755 ${HOME}/bin/lein
ENV LEIN_ROOT 1
RUN lein
ADD dotfiles/.lein/profiles.clj ${HOME}/.lein/

USER root
ENV HOME /root
WORKDIR /root

ADD sv /etc/service
CMD ["/sbin/my_init"]
RUN apt-get clean && rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*
```