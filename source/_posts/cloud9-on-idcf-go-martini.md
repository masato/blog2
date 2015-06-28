title: "Cloud9 on Docker - Part3: GoとMartiniをインストール"
date: 2015-06-24 05:17:49
tags:
 - Cloud9
 - Go
 - Martini
 - IDCFクラウド
description: じぶんクラウドのDocker上に構築したCloud9にブラウザら接続してどこでもNode.jsの開発ができるようになりました。メニューのRun  Run with にはデフォルトでいろいろな言語のRnnerがあります。ここにGoもありました。最近使っていなかったので今回はGoの開発環境を追加しようと思います。
---

じぶんクラウドのDocker上に構築したCloud9にブラウザら接続してどこでもNode.jsの開発ができるようになりました。メニューのRun - Run with にはデフォルトでいろいろな言語のRnnerがあります。ここにGoもありました。最近使っていなかったので今回はGoの開発環境を追加しようと思います。

<!-- more -->

## プロジェクト

デフォルトのNode.jsに加えてGoをインストールします。リポジトリは[こちら](https://github.com/masato/docker-cloud9/tree/go)です。Cloud9へのGoのインストールについては[Writing a Go App](https://docs.c9.io/v1.0/docs/writing-a-go-app)に書いてあります。
 
### Dockerfile

Goをダウンロードしてインスト-ルします。Go環境変数は`/root/.profile`に追記します。

```~/node_apps/docker-cloud9/Dockerfile
FROM node:0.12
MAINTAINER Masato Shimizu <ma6ato@gmail.com>

RUN git clone https://github.com/c9/core.git /cloud9 && \
    cd /cloud9 && ./scripts/install-sdk.sh

RUN npm install hexo-cli -g
RUN wget -O - https://storage.googleapis.com/golang/go1.4.2.linux-amd64.tar.gz | tar -xzC /usr/local -f - &\
& \
    echo "export GOPATH=/workspace/gocode" >> /root/.profile && \
    echo "export PATH=$PATH:/usr/local/go/bin:/workspace/gocode/bin" >> /root/.profile

WORKDIR /workspace
```

### docker-compose.yml

docker-compose.ymlには今回使用するWebアプリの[Martini](https://github.com/go-martini/martini)で使用する3000ポートを公開します。

```~/node_apps/docker-cloud9/docker-compose.yml
cloud9:
  build: .
  restart: always
  ports:
    - 8080:80
    - 15454:15454
    - 3000:3000
    - 4000:4000
  volumes:
    - ./workspace:/workspace
    - ~/.gitconfig:/root/.gitconfig:ro
    - ~/.ssh/id_rsa:/root/.ssh/id_rsa:ro
    - /etc/localtime:/etc/localtime:ro
  command: node /cloud9/server.js --port 80 -w /workspace --auth user:password
```

Dockerイメージを再作成します。これまでNode.jsのアプリは`/workspace`で作業していました。このディレクトリはDockerホストにマップしているので起動しているコンテナやイメージを破棄しても消えません。

```bash
$ docker-compose build
$ docker-compose up -d
```

Cloud9のコンソールから環境変数が設定されているか確認します。

```bash
$ go env GOROOT
/usr/local/go
$ go env GOPATH
/workspace/gocode
$ which go
/usr/local/go/bin/go
```

## Martini

[Martini](https://github.com/codegangsta/martini)はGo用のSinatraライクなマイクロフレームワークです。Cloud9のコンソールから`go get`でインストールします。インストール先は`GOPATH`に設定した`/workspace/gocode/`になります。


```bash
$ go get github.com/go-martini/martini
```


### server.go

workspaceに適当なディレクトリを作成します。`/`のルートと簡単なHello Worldを書きます。

```/workspace/martini/server.go
package main

import "github.com/go-martini/martini"

func main() {
  m := martini.Classic()
  m.Get("/", func() string {
    return "Hello world!"
  })
  m.Run()
}
```

### 起動

server.goを選択した状態でCloud9のRunボタンを押して起動します。

![martini-run.png](/2015/06/24/cloud9-on-idcf-go-martini/martini-run.png)

またはCloud9のコンソールから直接`go run`を実行します。

```bash
$ cd /workspace/martini
$ go run server.go
````

コンソールからcurlでテストします。

```bash
$ curl localhost:3000
Hello world!root
```

### Previewが動かない

メニューからPreview -> Preview Runnning Applicationを選択できます。残念ながらホスト名がundefinedになってしまいました。まだ設定が足りないようです。

![cloud9-preview-undefined.png](/2015/06/24/cloud9-on-idcf-go-martini/cloud9-preview-undefined.png)

直接Cloud9のホストに対してブラウザを開くとHello Worldが表示されました。とりあえずプレビューはできそうです。

![martini-hello.png](/2015/06/24/cloud9-on-idcf-go-martini/martini-hello.png)
