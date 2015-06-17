title: "Cloud9をIDCFクラウドで使う - Part1: Docker Composeでインストール"
date: 2015-06-08 10:16:48
tags:
 - Cloud9
 - Nodejs
 - IDCFクラウド
 - Docker
 - DockerCompose
description: Introducing Nitrous ProにあるようにすでにNitrous.IO Proに移行しています。暫定的に旧環境もLiteとして使えていますが、2015-07-01には終了します。Nitours Proのアカウントも作成しましたがfreeアカウントだと1日2時間しか使えません。そろそろ移行先を考えないといけません。
---

これまで[Nitrous.IO](https://pro.nitrous.io/)をfreemiumでありがたく使わせていただきましたが、[Introducing Nitrous Pro](http://blog.nitrous.io/2015/04/22/introducing-nitrous-pro.html)にあるようにすでにNitrous.IO Proに移行しています。暫定的に旧環境もLiteとして使えていますが、2015-07-01には終了します。Nitours Proのアカウントも作成しましたがfreeアカウントだと1日2時間しか使えません。そろそろ移行先を考えないといけません。

<!-- more -->

## Nitrous.IO

Nitrous.IOはWebIDEやCloudIDEと呼ばれるカテゴリーで、クラウド上に開発環境のコンテナととブラウザから使えるIDEがセットになったサービスです。alternativesには以下のサービスがあります。

* [Cloud9](https://c9.io/)
* [Codio](https://codio.com/)
* [Koding](https://koding.com/)
* [Codenvy](https://codenvy.com/)

主にNode.jsやGoの開発用に使っていますが、DjangoやRails、MEANやMeteorなどのテンプレートもそろっています。ブラウザがあればすぐに開発とプレビューがすぐに使えるようになります。Chromebookとネット接続環境さえあればどこでも開発ができるようになります。ホームレスからスーパースターまで。

2年くらい前に[After Just Four Weeks, The Homeless Man Learning To Code Has Almost Finished His First App](http://www.businessinsider.com/homeless-coder-2013-9)をみて思い立ちChromebookとNitrous.IOを使い始めました。[Nitrous.IO Stories - Yehuda Katz and Tilde.IO](http://blog.nitrous.io/2013/08/05/nitrous-stories-i-yehuda-katz-tilde-nitrousio.html)によると、あのYehuda Katz氏も[Tilde.IO](http://www.tilde.io/)で使っているそうです。

## CloueIEDとIaaS

Cloud9にはパブリックサービスの[c9.io](https://c9.io)に加えてオープンソース版の[c9/core](https://github.com/c9/core/)があります。オープンソース版をDockerコンテナとしてCloud9を起動する予定なのでどのIaaSでも動作します。Cloud9は[BeagleBone Black](http://beagleboard.org/)でも採用されているようにそれほど大きなリソースも必要ありません。今回はミニマム構成だと月額500円で使えてコスパが良い[IDCFクラウド](https://idcfcloud.com)で動かしてみます。

* Nitrous.IOの[BASIC](https://pro.nitrous.io/pricing/#standard)
 * コンテナ: 2つまで
 * メモリ: 1GB
 * ストレージ: 10GB
 * 料金: 14.99ドル/月
 
* c9.ioの[Micro](https://c9.io/web/site/pricing)
 * コンテナ: 2つまで
 * メモリ: 1GB
 * ストレージ: 10GB
 * 料金: 9ドル/月
 
* IDCFクラウドの[light.S1](http://www.idcf.jp/cloud/price.html)
 * コンテナ: ストレージとメモリの範囲内
 * メモリ: 1GB
 * ストレージ: 15GB
 * 料金: 500円/月
 
## プロジェクト

適当なディレクトリにプロジェクトを作成します。workspaceはCloud9のコンテナにマップするボリュームです。リポジトリは[こちら](https://github.com/masato/docker-cloud9)です。

```bash
$ cd ~/node_apps/docker-cloud9
$ tree -L 1
.
├── Dockerfile
├── docker-compose.yml
├── package.json
└── workspace
```

Docker Hub RegistryにいくつかCloud9のイメージがあります。

* [kdelfour/cloud9-docker/](https://registry.hub.docker.com/u/kdelfour/cloud9-docker/)
* [levanter69/cloud9-nodejs/dockerfile/](https://registry.hub.docker.com/u/levanter69/cloud9-nodejs/dockerfile/)

今回はDocker Composeを使ってミニマルにつくりたいので自分でビルドすることにします。

```bash ~/node_apps/docker-cloud9/Dockerfile
FROM node:0.12
MAINTAINER Masato Shimizu <ma6ato@gmail.com>

RUN git clone https://github.com/c9/core.git /cloud9 && \
    cd /cloud9 && ./scripts/install-sdk.sh

WORKDIR /workspace
```

以下のようなdocker-compose.ymlを用意します。portやcommandはこれからも変更していくのでリビルドに時間がかかるDockerイメージには入れませんでした。

```yaml ~/node_apps/docker-cloud9/docker-compose.yml
cloud9:
  build: .
  ports:
    - 8080:80
    - 15454:15454
    - 3000:3000
    - 5000:5000
  volumes:
    - ./workspace:/workspace
  command: node /cloud9/server.js --port 80 -w /workspace --auth user:password
```

Cloud9は80ポートで起動します。Dockerホストで80は他のサービスですでに使っているので8080にマップします。15454ポートはデバッグ用に使うポートです。5000ポートはテスト用のExpressアプリで使います。

## Cloud9

Docker Composeからサービスをupします。初回の起動時にイメージをビルドするので10分くらいがかかります。

```bash
$ docker-compose up
Recreating dockercloud9_cloud9_1...
Attaching to dockercloud9_cloud9_1
cloud9_1 | Connect server listening at http://172.17.3.166:80
cloud9_1 | Using basic authentication
cloud9_1 | CDN: version standalone initialized /cloud9/build
cloud9_1 | Started '/cloud9/configs/standalone' with config 'standalone'!
```

Dockerホストには8080でマップしているので、ブラウザからDockerホストのIPアドレスを指定して開きます。

http://xxx.xxx.xxx.xxx:8080/ide.html

![cloud9-init.png](/2015/06/08/cloud9-on-idcf-install/cloud9-init.png)

### Expressアプリの作成

`workspace`の下に新しいディレクトリを作成します。ミニマルなExpressのapp.jsを作成します。

```js app.js
'use strict';
var express = require('express');
var app = express();

app.get("/", function(req, res) {
    res.send("Hello, world!");
});

app.listen(5000);

console.log('Running on http://localhost:5000');
```

Expressをインストールしてapp.jsを実行するpackage.jsonを作成します。

```json package.json
{
 "name": "express-minimal",
  "description": "express-minimal",
  "version": "0.0.1",
  "private": true,
  "dependencies": {
    "express": "^4.12.4"
  },
  "scripts": {"start": "node app.js"}
}
```

IDE上では以下のようになります。

![cloud9-spike.png](/2015/06/08/cloud9-on-idcf-install/cloud9-spike.png)

### 実行

画面下のコンソールから作成した`spike`ディレクトリに移動してnpmパッケージをインストールします。

```bash
root@744ec8fcf715:/workspace# cd spike/
root@744ec8fcf715:/workspace/spike# npm install
```

app.jsを選択した状態で、メニューから`Run > Run With > Node.js`を選択してアプリを起動します。新しいコンソールタブでデバッグ用の15454ポートと、Expressの5000ポートが開きます。

![cloud9-runwith.png](/2015/06/08/cloud9-on-idcf-install/cloud9-runwith.png)

```bash
Debugger listening on port 15454
Running on http://localhost:5000
```

ExpressアプリでHello worldが表示されました。とりあえず動いているようです。

![cloud9-express.png](/2015/06/08/cloud9-on-idcf-install/cloud9-express.png)
