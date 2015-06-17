title: "Dockerコンテナ上でCLIを実行するデザインパターン"
date: 2015-04-22 09:40:58
categories:
 - Docker
tags:
 - Docker
 - DockerCompose
 - ember-cli
description: サーバーはDockerコンテナで起動することが多くなりました。サーバーと一緒に使うクライアントもDockerイメージで配布できるとホストマシンの環境も汚さないので便利です。いつくか参考になるDockerfileを見比べながらよいデザインパターンを考えていこうと思います。
---

サーバーはDockerコンテナで起動することが多くなりました。サーバーと一緒に使うクライアントもDockerイメージで配布できるとホストマシンの環境も汚さないので便利です。いつくか参考になるDockerfileを見比べながらよいデザインパターンを考えていこうと思います。

<!--more-->

## nsenter - Dockerホストにコピーしてインストール

[nsenter](https://github.com/jpetazzo/nsenter/)はDockerイメージのビルドでバイナリをビルドします。コンテナを起動してからDockerホストにコピーしてインストールしています。

```bash Dockerfile
FROM debian:jessie
ENV VERSION 2.26
RUN apt-get update -q
RUN apt-get install -qy curl build-essential
RUN mkdir /src
WORKDIR /src
RUN curl https://www.kernel.org/pub/linux/utils/util-linux/v$VERSION/util-linux-$VERSION.tar.gz \
     | tar -zxf-
RUN ln -s util-linux-$VERSION util-linux
WORKDIR /src/util-linux
RUN ./configure --without-ncurses
RUN make LDFLAGS=-all-static nsenter
RUN cp nsenter /
ADD docker-enter /docker-enter
ADD installer /installer
CMD /installer
# Now build the importenv helper
WORKDIR /src
ADD importenv.c /src/importenv.c
RUN make LDFLAGS=-static CFLAGS=-Wall importenv
RUN cp importenv /
```

とてもきれいなワンライナーインストールです。[nsenter](https://github.com/jpetazzo/nsenter/)は`docker exec`が実装される前はよく使っていました。

``` bash
$ docker run --rm jpetazzo/nsenter cat /nsenter > /tmp/nsenter && chmod +x /tmp/nsenter
```

## tutum-cli - ENTRYPOINTを使う

[tutum-cli](https://github.com/tutumcloud/tutum-cli)はDockerイメージのビルドでソースコードから`pip install`しています。

```bash Dockerfile
FROM tutum/curl
MAINTAINER Tutum <info@tutum.co>

RUN apt-get update && \
    apt-get install -y python python-dev python-pip libyaml-dev
ADD . /app
RUN export SDK_VER=$(cat /app/requirements.txt | grep python-tutum | grep -o '[0-9.]*') && \
    curl -0L https://github.com/tutumcloud/python-tutum/archive/v${SDK_VER}.tar.gz | tar -zxv && \
    pip install python-tutum-${SDK_VER}/. && \
    pip install /app && \
    rm -rf /app python-tutum-${SDK_VER} && \
    tutum -v

ENTRYPOINT ["tutum"]
```

ENTRYPOINTにtutumコマンドを指定しています。コンテナを起動するときはtutumのサブコマンドを引数にします。

``` bash
$ docker run -e TUTUM_USER=username -e TUTUM_APIKEY=apikey tutum/cli service
```

dockerのコマンドが長くなるので`~/.bashrc`などにaliasを定義しておきます。

``` bash
alias tutum="docker run -e TUTUM_USER=username -e TUTUM_APIKEY=apikey tutum/cli"
tutum service
```

## google-cloud-sdk - VOLUMEコンテナを使う

google-cloud-sdkは機能が多いので依存するパッケージもたくさんあります。[google/cloud-sdk](https://registry.hub.docker.com/u/google/cloud-sdk/)を使うとホストマシンを汚さず、CLI環境をコンテナに閉じることができます。

```bash Dockerfile
FROM google/debian:wheezy
ENV DEBIAN_FRONTEND noninteractive
RUN apt-get update && apt-get install -y -qq --no-install-recommends wget unzip python php5-mysql php5-cli php5-cgi openjdk-7-jre-headless openssh-client python-openssl && apt-get clean
RUN wget https://dl.google.com/dl/cloudsdk/release/google-cloud-sdk.zip && unzip google-cloud-sdk.zip && rm google-cloud-sdk.zip
ENV CLOUDSDK_PYTHON_SITEPACKAGES 1
RUN google-cloud-sdk/install.sh --usage-reporting=true --path-update=true --bash-completion=true --rc-path=/.bashrc --disable-installation-options
RUN google-cloud-sdk/bin/gcloud --quiet components update pkg-go pkg-python pkg-java preview app
RUN google-cloud-sdk/bin/gcloud --quiet config set component_manager/disable_update_check true
RUN mkdir /.ssh
ENV PATH /google-cloud-sdk/bin:$PATH
ENV HOME /
VOLUME ["/.config"]
CMD ["/bin/bash"]
```

最初にgcloud-configボリュームコンテナを作成します。ブラウザから取得した識別コードなど認証情報を保存しておきます。

``` bash
$ docker run -t -i --name gcloud-config google/cloud-sdk gcloud auth login
```

ENTRYPOINTはバイナリに指定されていません。gcutilを引数にしてコンテナを起動します。認証情報はgcloud-configボリュームコンテナをマウントして使います。

``` bash
$ docker run --rm -ti --volumes-from gcloud-config google/cloud-sdk gcutil listinstances
```

## ember-cli - Docker Composeを使う

Docker Compose(Fig)をつかったパターンです。google-cloud-sdkと同じようにコンテナ間で共通のボリュームにデータを保存します。 [geoffreyd/ember-cli](https://registry.hub.docker.com/u/geoffreyd/ember-cli/)はFigを使っていますが、Docker Composeでも動作します。アプリ実行に必要なサーバー、DB、クライアントのイメージをDocker Composeで配布することを考えているのでこのパターンが一番よさそうです。

``` bash Dockerfile
FROM node

RUN npm install -g ember-cli@0.1.0 bower

EXPOSE 4200 35729
WORKDIR /usr/src/app
ENTRYPOINT ["/usr/local/bin/ember"]
CMD ["help"]
```

### docker-compose.yml

Rubyなどでよく見るYAMLのマージをつかっています。Docker Compose 1.2から別のYAMLファイルを読み込む[Extends](https://docs.docker.com/compose/extends/)書式が追加されましたが、コンパクトに1ファイルにまとめるならYAMLのマージでもよさそうです。

```yaml ~/compose_apps/docker-compose.yml
ember: &defaults
  image: geoffreyd/ember-cli
  volumes:
    - .:/usr/src/app

server:
  <<: *defaults
  command: server --watcher polling
  ports:
    - 4200:4200
    - 35729:35729

npm:
  <<: *defaults
  entrypoint: ['/usr/local/bin/npm']

bower:
  <<: *defaults
  entrypoint: ['/usr/local/bin/bower', '--allow-root']
```

### ember-cliの簡単な使い方

ember-cliをつかった簡単なサンプルを書いてみます。コンテナ内のemberコマンドから生成したァイルはroot権限になります。Dockerホストからボリュームにマウントしたディレクトリ（今回はカレントディレクトリ）のファイルを直接編集する場合もroot権限が必要になるため、パーミッションの変更などちょっと工夫が必要なようです。

ember-cliコマンドを使ってemberのプロジェクトを作成します。

``` bash
$ docker-compose run --rm ember new ember-sample
...
Installed packages for tooling via npm.
Installed browser packages via Bower.
Successfully initialized git.
```

作成したemberプロジェクトに移動して、docker-compose.ymlとpackage.jsonのテストをします。

``` bash
$ sudo mv docker-compose.yml ember-sample
$ cd ember-sample
$ docker-compose run --rm npm install
```

bowerのbootstrapをインストールします。

``` bash
$ docker-compose run --rm bower install bootstrap
bower not-cached    git://github.com/twbs/bootstrap.git#*
bower resolve       git://github.com/twbs/bootstrap.git#*
bower download      https://github.com/twbs/bootstrap/archive/v3.3.4.tar.gz
bower extract       bootstrap#* archive.tar.gz
bower resolved      git://github.com/twbs/bootstrap.git#3.3.4
bower install       bootstrap#3.3.4

bootstrap#3.3.4 bower_components/bootstrap
└── jquery#1.11.3
```

emberのモデルを作成します。

``` bash
$ docker-compose run --rm ember generate model user
version: 0.1.0
installing
  create app/models/user.js
installing
  create tests/unit/models/user-test.js
Removing embersample_ember_run_1...
```

server用のコンテナを起動してブラウザから確認します。

``` bash
$ docker-compose up server
Creating embersample_server_1...
Attaching to embersample_server_1
server_1 | version: 0.1.0
server_1 | Livereload server on port 35729
server_1 | Serving on http://0.0.0.0:4200
server_1 |
server_1 | Build successful - 1213ms.
server_1 |
server_1 | Slowest Trees                  | Total
server_1 | -------------------------------+----------------
server_1 | Concat                         | 496ms
server_1 | TemplateCompiler               | 139ms
server_1 | ES6Concatenator                | 84ms
server_1 | JSHint - App                   | 73ms
server_1 |
```

![ember-sample.png](/2015/04/22/docker-container-cli-pattern/ember-sample.png)