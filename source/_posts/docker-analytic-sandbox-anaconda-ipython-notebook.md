title: "Dockerでデータ分析環境 - Part2: AnacondaでIPython Notebookをインストール"
date: 2014-07-11 23:20:11
tags:
 - Docker
 - AnalyticSandbox
 - Ubuntu
 - IPythonNotebook
 - R
 - Anaconda
 - nbserver
 - Druid
 - runit
description: Part1でRStudio Serverをインストールしたのに続いて、IPython Notebook環境を構築します。マルチユーザー環境だと、ipython-hydraがありますが理解できていないので、rootユーザーで動かすことにします。IACSのコースでは、IPython Notebookの環境構築にAnacondaを使っているので参考にします。AnacondaはPythonでデータ分析環境を構築する場合に必要なパッケージがたくさん入っているので、まとめてインストールできます。
---

インストールの手順を間違えていたので更新しました。Nginx+SSLのリバースプロキシを使うと、MathJaxのインストールがCDNからできないようです。事前にローカルにインストールするようにしました。

* `Update 2014-08-10`: [Dockerでデータ分析環境 - Part7: Ubuntu14.04のDockerにIPython Notebookをデプロイ](/2014/08/10/docker-analytic-sandbox-ipython-notebook-deploy/)

[Part1](/2014/07/09/docker-analytic-sandbox-rstudio-server/)で`RStudio Server`をインストールしたのに続いて、`IPython Notebook`環境を構築します。
マルチユーザー環境だと、[ipython-hydra](https://github.com/cni/ipython-hydra)がありますが理解できていないので、rootユーザーで動かすことにします。

[IACS](http://iacs-courses.seas.harvard.edu/courses/am207/blog/installing-python.html)のコースでは、`IPython Notebook`の環境構築に[Anaconda](https://store.continuum.io/cshop/anaconda/)を使っているので参考にします。
AnacondaはPythonでデータ分析環境を構築する場合に必要なパッケージがたくさん入っているので、まとめてインストールできます。

<!-- more -->

### Dockerfile

[phusion/baseimage](https://registry.hub.docker.com/u/phusion/baseimage/)を使います。
昨日の`RStudio Server`用のDockerfileに追加します。

``` bash ~/docker_apps/rstudio-server/Dockerfile
FROM phusion/baseimage:0.9.11

ENV HOME /root
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
ENV EDITOR vim
RUN update-alternatives --set editor /usr/bin/vim.basic
RUN DEBIAN_FRONTEND=noninteractive \
    apt-get install -y git wget curl unzip

## Install R
RUN echo 'deb http://cran.rstudio.com/bin/linux/ubuntu trusty/' > /etc/apt/sources.list.d/r.list && \
    apt-key adv --keyserver keyserver.ubuntu.com --recv-keys E084DAB9 && \
    apt-get update
RUN DEBIAN_FRONTEND=noninteractive apt-get install -y r-base

## RStudio Server
RUN DEBIAN_FRONTEND=noninteractive apt-get install -y gdebi-core libapparmor1 libcurl4-openssl-dev
RUN wget http://download2.rstudio.org/rstudio-server-0.98.953-amd64.deb && \
    gdebi -n rstudio-server-0.98.953-amd64.deb && \
    rm rstudio-server-*-amd64.deb
RUN useradd -m rstudio && \
    echo "rstudio:rstudio" | chpasswd

## Anaconda
RUN wget http://09c8d0b2229f813c1b93-c95ac804525aac4b6dba79b00b39d1d3.r79.cf1.rackcdn.com/Anaconda-2.0.1-Linux-x86_64.sh -P /root \
  && yes | bash /root/Anaconda-2.0.1-Linux-x86_64.sh

RUN . /root/.bashrc && ipython profile create default \
  && mkdir /notebook

ADD sv /etc/service

# Use baseimage-docker's init system.
CMD ["/sbin/my_init"]

# Clean up APT when done.
RUN apt-get clean && rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*
```

### runitの設定ファイル

Anacondaではipythonなど実行ファイルは、`/y/bin/`にインストールされます。

これはyesコマンドでパイプしていることが原因でした。rootで`Anaconda-2.0.1-Linux-x86_64.sh -b`のように`-b`フラグをつけて実行すると`/root/anaconda/`ディレクトリにインストールされます。

* `Update 2014-08-10`: [Dockerでデータ分析環境 - Part7: Ubuntu14.04のDockerにIPython Notebookをデプロイ](/2014/08/10/docker-analytic-sandbox-ipython-notebook-deploy/)


``` bash ~/docker_apps/rstudio-server/sv/ipython/run
#!/bin/sh

exec 2>&1
exec /y/bin/ipython notebook --no-browser --profile nbserver --pylab=inline --ip=0.0.0.0 --port 8080 --notebook-dir=/notebook
```

svlogdを使ったログの設定です。

``` bash ~/docker_apps/rstudio-server/sv/rstudio-server/log/run
#!/bin/sh
service=$(basename $(dirname $(pwd)))
logdir="/var/log/${service}"
mkdir -p ${logdir}
exec 2>&1
exec /usr/bin/svlogd -tt ${logdir}
```

### イメージの build と run

イメージをビルドします。

``` bash
$ cd ~/docker_apps/rstudio-server
$ docker build -t masato/rstudio-server . 
```

コンテナを起動します。SSHと`RStudio Server`のポートをエクスポーズします。

``` bash
$ docker run --rm -i -t -p 1022:22 -p 8788:8787 -p 8088:8080 masato/rstudio-server  /sbin/my_init bash
```

### ブラウザで確認

Dockerホストからブラウザを開いて、`IPython Notebook`を使ってみます。

{% img center /2014/07/11/docker-analytic-sandbox-anaconda-ipython-notebook/ipython-notebook.png %}

### まとめ

`RStudio Server`に追加して、マルチユーザー機能はまだ課題ですが、`IPython Notebook`も使えるようになりました。
次はPostgreSQLの環境を用意しようと思います。

