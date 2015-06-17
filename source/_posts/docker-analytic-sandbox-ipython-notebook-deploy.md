title: "Dockerでデータ分析環境 - Part7: Ubuntu14.04のDockerにIPython Notebookをデプロイ"
date: 2014-08-10 09:21:37
tags:
 - Docker
 - AnalyticSandbox
 - Ubuntu
 - IPythonNotebook
 - Anaconda
 - runit
 - OpenResty
description: ローカルで動いていたIPython NotebookのコンテナをOpenRestyのリバースプロキシの下に配置したところ、
IPythonでNotebookの作成に失敗してしまいました。Dockerでデータ分析環境 - Part2: AnacondaでIPython Notebookをインストールで作ったDockerファイルと、Dockerでデータ分析環境 - Part6 Ubuntu14.04のDockerにRStudio ServerとOpenRestyをデプロイで作成したnginx.confを修正していきます。
---

* `Update 2014-09-29`: [Dockerデータ分析環境 - Part8: Gorilla REPLとCMDのcdでno such file or directoryになる場合](/2014/09/29/docker-analytic-sandbox-gorilla-repl/)

ローカルで動いていた`IPython Notebook`のコンテナを、OpenRestyのリバースプロキシの下に配置したところ、IPythonでNotebookの作成に失敗してしまいました。

[Dockerでデータ分析環境 - Part2: AnacondaでIPython Notebookをインストール](/2014/07/11/docker-analytic-sandbox-anaconda-ipython-notebook/)で作ったDockerファイルと、[Dockerでデータ分析環境 - Part6: Ubuntu14.04のDockerにRStudio ServerとOpenRestyをデプロイ](/2014/08/09/docker-analytic-sandbox-rstudio-openresty-deploy/)で作成したnginx.confを修正していきます。

<!-- more -->


### OpenRestyのnginx.conf

`IPython Noteook`ではWebSocketを利用しているため、nginx.confにWebSocketサポートを追加します。
[ipython-notebook-cookbook](https://github.com/rgbkrk/ipython-notebook-cookbook/blob/master/templates/default/nginx-proxy.erb)が作成するNginxの設定ファイルを参考にしました。

``` bash ~/docker_apps/openresty/nginx.conf
...
        ### By default we don't want to redirect it ####
        proxy_redirect     off;
        
        proxy_set_header X-NginX-Proxy true;

        # WebSocket support
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_read_timeout 86400;

        location / {
            set $upstream "";        
...
```

OpenRestyのイメージもビルドし直して、本番環境で入れ替えます。

### IPython NotebookのDockerfile

`RStudio Server`と`IPython Notebook`は同じDockerfileにしていましたが、今回から分離して使います。
これまでyesコマンドをパイプしていたのでインストール先も`/y/`ディレクトリになっていました。
rootで`Anaconda-2.0.1-Linux-x86_64.sh -b`のように`-b`フラグをつけて実行すると`/root/anaconda/`ディレクトリにサイレントインストールできました。


``` bash ~/docker_apps/ipython/Dockerfile
FROM phusion/baseimage:0.9.11
MAINTAINER Masato Shimizu <ma6ato@gmail.com>

ENV HOME /root
RUN /etc/my_init.d/00_regen_ssh_host_keys.sh

## Install an SSH of your choice.
ADD google_compute_engine.pub /tmp/your_key
RUN cat /tmp/your_key >> /root/.ssh/authorized_keys \
  && chmod 600 /root/.ssh/authorized_keys \
  && chmod 600 /root/.ssh/authorized_keys && rm -f /tmp/your_key

## ...put your own build instructions here...
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

## Anaconda
RUN wget http://09c8d0b2229f813c1b93-c95ac804525aac4b6dba79b00b39d1d3.r79.cf1.rackcdn.com/Anaconda-2.0.1-Linux-x86_64.sh -P /root \
  && /bin/bash /root/Anaconda-2.0.1-Linux-x86_64.sh -b
RUN /root/anaconda/bin/ipython profile create nbserver && mkdir /notebook
RUN /root/anaconda/bin/ipython -c 'from IPython.external import mathjax; mathjax.install_mathjax(tag="2.2.0")'

ADD sv /etc/service

# Use baseimage-docker's init system.
CMD ["/sbin/my_init"]

# Clean up APT when done.
RUN apt-get clean && rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*
```

runitの設定ファイルも、`--pylab=inline`がdeprecatedなので削除しました。

``` bash ~/docker_apps/ipython/sv/ipython/run
#!/bin/sh

exec 2>&1
exec /root/anaconda/bin/ipython notebook --no-browser --profile nbserver --ip=0.0.0.0 --port 8080 --notebook-dir=/notebook
```

なぜかNginx+SSLのリバースプロキシ下にいるとMathJaxがCDNからインストールされないようです。
また、mathjax.pyのtag引数のデフォルトは`v2.2`ですが、現在そのファイル名はないので、正しく`2.2.0`と指定する必要があります。
``` python ~/anaconda/lib/python2.7/site-packages/IPython/external/mathjax.py
def install_mathjax(tag='v2.2', dest=default_dest, replace=False, file=None, extractor=extract_tar):
    """Download and/or install MathJax for offline use.

    This will install mathjax to the nbextensions dir in your IPYTHONDIR.

    MathJax is a ~15MB download, and ~150MB installed.
```


### docker-registryへpushとpull

イメージをビルドして、docker-registryへpushします。

``` bash
$ docker build -t masato/ipython .
$ docker tag masato/ipython localhost:5000/ipython
$ docker push localhost:5000/ipython
```

本番環境のDockerホストにSSH接続をして、イメージをpullします。

```
$ docker pull 10.1.2.164:5000/ipython
```

コンテナを起動して、ルーティングを設定します。

``` bash
$ docker run -i -t -d --name ipython 10.1.2.164:5000/ipython /sbin/my_init
$ docker run -it --rm --link openresty:redis dockerfile/redis bash -c 'redis-cli -h $REDIS_PORT_6379_TCP_ADDR set masato-ipython.10.1.2.244.xip.io 172.17.0.96:8080'
```

### 確認

ブラウザで開いて確認します。WebSocketやMathJaxのエラーもなく、`New Notebook`が実行できました。

{% img center /2014/08/10/docker-analytic-sandbox-ipython-notebook-deploy/ipython-notebook.png %}




