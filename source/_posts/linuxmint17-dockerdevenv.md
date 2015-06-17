title: "LinuxMint17 に PythonのDocker開発環境をインストールする"
date: 2014-08-22 04:50:36
tags:
 - LinuxMint17
 - DockerDevEnv
 - Python
 - nsenter
description: あたらしくPythonで開発を始めることになったので、気分転換にLinx Mint 17 Qianaに開発環境を移動しました。Ubuntu 14.04 LTSがベースになっていますが、少し違うところがあるので編集していきます。Dartの開発用にIDCFクラウドでLinuxMint17 MATEをxrdpから使う - Part2 インストール編で作成していたインスタンスにDockerをインストールして使います。
---

あたらしくPythonで開発を始めることになったので、気分転換に`Linx Mint 17 Qiana`に開発環境を移動しました。
`Ubuntu 14.04 LTS`がベースになっていますが、少し違うところがあるので編集していきます。

Dartの開発用に[IDCFクラウドでLinuxMint17 MATEをxrdpから使う - Part2: インストール編](/2014/06/02/idcf-linuxmint17-part2/)で作成していたインスタンスにDockerをインストールして使います。

<!-- more -->

### Linx Mint 17の情報

`Linx Mint 17`もUbuntuと同様に`/etc/lsb-release`にリリース情報があります。

``` bash /etc/lsb-release
DISTRIB_ID=LinuxMint
DISTRIB_RELEASE=17
DISTRIB_CODENAME=qiana
DISTRIB_DESCRIPTION="Linux Mint 17 Qiana"
```

kernelのバージョンです。

``` bash
$ uname -r
3.13.0-24-generic
```

### GRUBの設定

"Your kernel does not support cgroup swap limit."と表示されるので、grubを編集します。

``` bash /etc/default/grub
GRUB_CMDLINE_LINUX="cgroup_enable=memory swapaccount=1" 
```
update-grubをします。

``` bash
$ sudo update-grub
Generating grub configuration file ...
Warning: Setting GRUB_TIMEOUT to a non-zero value when GRUB_HIDDEN_TIMEOUT is set is no longer supported.
Found linux image: /boot/vmlinuz-3.13.0-24-generic
Found initrd image: /boot/initrd.img-3.13.0-24-generic
Found memtest86+ image: /memtest86+.elf
Found memtest86+ image: /memtest86+.bin
完了
```

rebootします。

``` bash
$ sudo reboot
```

### Dockerのインストール

Dockerのインストールは、Ubuntu14.04と同じです。最新版をインストールするため`docker.io`パッケージは使いません。

``` bash
$ sudo apt-get update
$ sudo apt-get upgrade
$ sudo apt-get install -y apt-transport-https apparmor
$ echo "deb https://get.docker.io/ubuntu docker main" | sudo tee -a  /etc/apt/sources.list.d/docker.list
$ sudo apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv-keys 36A1D7869245C8950F966E92D8576A8BA88D21E9
$ sudo apt-get update
$ sudo apt-get install -y lxc-docker
```

version

``` bash
$ docker -v
Docker version 1.1.2, build d84a070
```

sudo なしでdockerコマンドを使えるようにします。

``` bash
$ sudo usermod -aG docker $USER
$ exit
```

### nsenterのインストール

インストール

``` bash
$ docker run --rm -v /usr/local/bin:/target jpetazzo/nsenter
Installing nsenter to /target
Installing docker-enter to /target
```

~/bin/nse

``` bash ~/bin/nse
[ -n "$1" ] && sudo nsenter --pid --uts --mount --ipc --net --target $(docker inspect --format="&#123;&#123; .State.Pid &#125;&#125;" $1)
```

chmodで実行権限をつけます。

``` bash
$ chmod u+x ~/bin/nse
```

### Python開発環境

``` bash ~/docker_apps/python/Dockerfile
FROM phusion/baseimage:0.9.12
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

## Development Environment
RUN mkdir -p /root/.byobu
RUN update-alternatives --set editor /usr/bin/vim.basic

RUN DEBIAN_FRONTEND=noninteractive \
    apt-get install -y build-essential software-properties-common \
                       zlib1g-dev libssl-dev libreadline-dev libyaml-dev \
                       libxml2-dev libxslt-dev sqlite3 libsqlite3-dev \
                       emacs24-nox emacs24-el git byobu wget curl unzip tree

# Add Un-Root User
RUN useradd -m -d /home/docker -s /bin/bash docker \
 && echo "docker:docker" | chpasswd \
 && mkdir /home/docker/.ssh \
 && chmod 700 /home/docker/.ssh \
 && cp /root/.ssh/authorized_keys /home/docker/.ssh \
 && chmod 600 /home/docker/.ssh/authorized_keys \
 && chown -R docker:docker /home/docker/.ssh \
 && echo "export LANG=ja_JP.UTF-8" >> /home/docker/.profile \
 && echo "docker ALL=(ALL) NOPASSWD:ALL" >> /etc/sudoers

## Python
RUN DEBIAN_FRONTEND=noninteractive \
  apt-get install -y python python-pip python-dev

## dotfiles
RUN mkdir -p /root/tmp /home/docker/tmp
ADD dotfiles /root/
RUN cp -R /root/.byobu /home/docker/ \
 && cp -R /root/.emacs.d /home/docker/ \
 && cp /root/.gitconfig /home/docker/ \
 && cp /root/.vimrc /home/docker/ \
 && chown -R docker:docker /home/docker

CMD ["/sbin/my_init"]

RUN apt-get clean && rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*
```

イメージのビルド

``` bash
$ docker build -t masato/python-base .
```

コンテナの起動

``` bash
$ docker run -t -i --name pythonenv masato/python-base /sbin/my_init bash
```