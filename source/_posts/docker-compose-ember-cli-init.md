title: "Docker Composeを使ってEmber CLI開発環境を作る"
date: 2015-04-25 13:46:03
categories:
 - Docker
tags:
 - Docker
 - DockerCompose
 - Emberjs
 - ember-cli
 - Nodejs
 - CLI
description: 前回Sane Stackの開発環境をDocker Composeで作成しました。Ember CLIの動作がよくわからなかったので今回はEmber CLIだけで環境を用意しようと思います。
---

前回[Sane Stack](/2015/04/24/sane-stack-emberjs-salisjs-docker-compose-quickstart/)の開発環境をDocker Composeで作成しました。Ember CLIの動作がよくわからなかったので今回はEmber CLIだけで環境を用意しようと思います。

<!-- more -->

## docker-compose.yml

### buildを使うとマージ先でもビルドされる

[DockerのCLI環境構築](/2015/04/22/docker-container-cli-pattern/)でテストした[geoffreyd/ember-cli](https://github.com/geoffreyd/ember-cli-docker)はNode.jsもEmber CLIも古いので自分でビルドすることにします。ただしdocker-compose.ymlでビルドを指定するとYAMLのマージで拡張したサービスでもビルドが走りそれぞれDockerイメージができてしまいます。

```yaml ~/compose_apps/docker-compose.yml
ember: &defaults
  build: .
  volumes:
    - $PWD:/usr/src/app
server:
  <<: *defaults
  command: server --watcher polling
  ports:
    - 4200:4200
    - 35729:35729
...s
```

以下のように予めローカルにビルドしておいたDockerイメージを指定します。

```yaml ~/compose_apps/docker-compose.yml
ember: &defaults
  image: ember-cli
  volumes:
    - $PWD:/emberapp

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

## Dockerfile

ローカルにDockerfileをビルドしておき、docker-compose.ymlではimgageで指定します。Node.jsはv0.12、ember-cliは最新の0.2.3を使います。[Watchman](https://facebook.github.io/watchman/)も追加しました。

``` bash ~/ember_apps/Dockerfile
FROM node:0.12
MAINTAINER Masato Shimizu <ma6ato@gmail.com>

# Watchman install
RUN git clone https://github.com/facebook/watchman.git && \
  cd watchman && \
  ./autogen.sh && \
  ./configure && \
  make && \
  make install

RUN npm install -g bower phantomjs ember-cli

# Create local user for development
RUN adduser --disabled-password --gecos '' --uid 1000 docker && \
  adduser docker sudo && \
  echo '%sudo ALL=(ALL) NOPASSWD:ALL' >> /etc/sudoers && \
  mkdir /emberapp && \
  chown docker:docker /emberapp
USER docker
WORKDIR /emberapp
EXPOSE 4200 35729
ENTRYPOINT ["/usr/local/bin/ember"]
CMD ["help"]
```


### 開発ユーザーの作成

ホストマシンのディレクトリをDockerコンテナにマウントしてファイルを作成した場合、通常はrootユーザー権限になります。ホストマシンではEmacsを使って作業ユーザーで開発をしたいのでこれではちょっと困ります。Dockerfileで作業ユーザーを同じUIDのユーザーを作成します。

``` bash
$ id -u
1000
```

開発用にUIDが1000のユーザーをDockerfileで作成します。

### WORKDIRの作成

ホストマシンにログインしている作業ユーザーのuidを確認します。Emberアプリを配置するWORKDIRは`/emberapp`を作成しました。これは`ember init`するディレクトリの名前が`app`だとサーバーを起動するときにエラーが発生するためです。

```bash 
$ docker-compose up server
Creating emberapps_server_1...
Attaching to emberapps_server_1
server_1 | version: 0.2.3
server_1 | 0.2.3
server_1 |
server_1 | Style file cannot have the name of the application - app
emberapps_server_1 exited with code 1
Gracefully stopping... (press Ctrl+C again to force)
Style file cannot have the name of the application - app
```

### Dockerイメージのビルド

あらかじめDockerイメージはローカルにビルドしておきます。

``` bash
$ docker build -t ember-cli .
```

## 使い方

最初にEmber CLIの初期処理を行います。Dockerfileではカレントディレクトリを`$PWD`でコンテナにマウントしています。カレントディレクトリにDockerfile、docker-compose.yml、package.jsonなど開発に必要なファイルを配置した状態にします。

``` bash
$ docker-compose run --rm ember init
```

BowerからBootstrapをインストールします。

``` bash
$ docker-compose run --rm ember install:bower bootstrap
```

[Asset Compilation](http://www.ember-cli.com/#asset-compilation)のページにあるようにBroccoliが`vendor.css`にプリコンパイルできるようにBootstrapのCSSを追加します。

``` js ~/ember_apps/Brocfile.js
app.import('bower_components/bootstrap/dist/css/bootstrap.css');
```

サーバーを起動します。

``` bash
$ docker-compose up server
Recreating emberapps_server_1...
Attaching to emberapps_server_1
server_1 | version: 0.2.3
server_1 | 0.2.3
server_1 |
server_1 | Livereload server on port 35729
server_1 | Serving on http://localhost:4200/
server_1 |
server_1 | Build successful - 3920ms.
server_1 |
server_1 | Slowest Trees                                 | Total
server_1 | ----------------------------------------------+---------------------
server_1 | Concat: Vendor                                | 3135ms
server_1 | Babel                                         | 228ms
server_1 |
server_1 | Slowest Trees (cumulative)                    | Total (avg)
server_1 | ----------------------------------------------+---------------------
server_1 | Concat: Vendor (1)                            | 3135ms
server_1 | Babel (2)                                     | 276ms (138 ms)
server_1 |
```
