title: "DockerでDartのRedstone.dartをはじめました"
date: 2014-05-18 02:07:51
tags:
 - Docker
 - Dockerfile
 - Dart
 - Redstone
 - Ubuntu
 - HelloWorld
description: IDCFクラウド上に構築したDockerに、Dartの開発環境を作りました。PythonのFlaskに似た、Redstone.dartというマイクロフレームワークで簡単なサンプルを作ってみます。
---
IDCFクラウド上に構築した[Docker](/2014/05/17/idcf-docker/)に、[Dart](https://www.dartlang.org/)の開発環境を作りました。
Pythonの[Flask](http://flask.pocoo.org/)に似た、[Redstone.dart](http://luizmineo.github.io/redstone.dart/)というマイクロフレームワークで簡単なサンプルを作ってみます。

<!-- more -->

### コンテナの起動

今回も[phusion/baseimage](https://github.com/phusion/baseimage-docker)のイメージを使いコンテナを起動します。
``` bash
$ docker run phusion/baseimage:latest /sbin/my_init --enable-insecure-key
...
No SSH host key available. Generating one...
Creating SSH2 RSA key; this may take some time ...
Creating SSH2 DSA key; this may take some time ...
Creating SSH2 ECDSA key; this may take some time ...
Creating SSH2 ED25519 key; this may take some time ...
invoke-rc.d: policy-rc.d denied execution of restart.
*** Running /etc/rc.local...
*** Booting runit daemon...
*** Runit started as PID 108
```

別シェルを起動して、コンテナのIPアドレスを確認します。[jq](http://stedolan.github.io/jq/)というJSONパーサーが便利そうなので一緒に使ってみます。
``` bash
$ docker inspect $(docker ps -a -q) | jq -r '.[0] | .NetworkSettings | .IPAddress'
172.17.0.2
```

SSH接続に使う、insecure_keyをダウンロードします。
``` bash
$ curl -o insecure_key -fSL https://github.com/phusion/baseimage-docker/raw/master/image/insecure_key
$ chmod 600 insecure_key
```
 
コンテナにSSH接続をします。
``` bash
$ ssh root@$172.17.0.2 -i ./insecure_key
Warning: Permanently added '172.17.0.2' (ECDSA) to the list of known hosts.
root@df1713debb18:~#
```

### Dart SDKのインストール

SSH接続したあと、必要なパッケージをインストールします。
``` bash
# apt-get update  
# apt-get install subversion make g++ openjdk-6-jdk git wget zip
```

Dartのコードをダウンロードするための[depot_tools](http://dev.chromium.org/developers/how-tos/install-depot-tools)をcloneします。

``` bash
# git clone http://chromium.googlesource.com/chromium/tools/depot_tools.git
# vi .profile
export PATH="$PATH":~/depot_tools
```

[Dart SDK](https://www.dartlang.org/tools/sdk/)をダウンロードします。PATHを通すため.profileを読み直します。
``` bash
# wget http://storage.googleapis.com/dart-archive/channels/stable/release/latest/sdk/dartsdk-linux-x64-release.zip
# unzip dartsdk-linux-x64-release.zip
# vi ~/.profile
export PATH="$PATH":~/depot_tools:~/dart-sdk/bin
# source ~/.profile
```

Dartのバージョンを確認します。
``` bash
# which dart
/root/dart-sdk/bin/dart
# dart --version
Dart VM version: 1.3.6 (Tue Apr 29 12:40:24 2014) on "linux_x64"
```

### Dockerイメージを作成する

`Dart SDK`のインストールができた状態で、一度イメージ作成します。その前にaptをきれいにします。
``` bash
# apt-get clean && rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*
# exit
```

Dockerホストに戻り、イメージを作成します。
``` bash
$ docker commit $(docker ps -a -q) masato/dart-sdk
```

作業していたコンテナを削除します。
``` bash
$ docker stop $(docker ps -a -q)
$ docker rm $(docker ps -a -q)
```

### 再度コンテナを起動する

Redstone.dartで使う8080ポートをフォワードしてコンテナを起動します。
``` bash
$ docker run -d -p 8080:8080  masato/dart-sdk:latest /sbin/my_init --enable-insecure-key
```

8080のポートフォワードを確認します。
``` bash
$ docker ps
CONTAINER ID        IMAGE                    COMMAND                CREATED              STATUS              PORTS                    NAMES
c3a2f4771f3d        masato/dart-sdk:latest   /sbin/my_init --enab   About a minute ago   Up About a minute   0.0.0.0:8080->8080/tcp   trusting_lalande
```

コンテナにSSH接続します。
``` bash
$ ssh root@172.17.0.2 -i ./insecure_key
 Warning: Permanently added '172.17.0.2' (ECDSA) to the list of known hosts.
Last login: Sat May 17 16:04:02 2014 from 172.17.42.1
root@c3a2f4771f3d:~#
```

### Redstone.dartをインストール

プロジェクトを作成します。
``` bash
# mkdir -p ~/dart_apps/my_app
# cd !$
```

pubコマンドがが依存関係をダウンロードするために、pubspec.yamlファイルを作成します。
``` yaml  ~/dart_apps/my_app/pubspec.yaml
name: my_app
dependencies:
  redstone: any
```

`pub get`で依存しているパッケージを団ロードします。
``` bash
# pub get
Resolving dependencies... (17.8s)
Downloading redstone 0.4.0...
Downloading crypto 0.9.0...
Downloading di 0.0.40...
Downloading grinder 0.5.2...
Downloading http_server 0.9.2...
Downloading route_hierarchical 0.4.20...
Downloading code_transformers 0.1.3...
Downloading analyzer 0.13.6...
Downloading path 1.1.0...
Downloading barback 0.12.0...
Downloading args 0.10.0+2...
Downloading quiver 0.18.2...
Downloading mime 0.9.0+1...
Downloading browser 0.10.0+2...
Downloading source_maps 0.9.0...
Downloading collection 0.9.2...
Downloading stack_trace 0.9.3+1...
Got dependencies!
```

### Dartプログラム

Flaskみたいな、簡単なDartプログラムを作成します。
``` dart ~/dart_apps/my_app/server.dart
import 'package:redstone/server.dart' as app;

@app.Route("/")
helloWorld() => "Hello, World!";

main() {
  app.setupConsoleLog();
  app.start();
}
```

サーバーを起動します。
``` bash
# cd ~/dart_apps/my_app
# dart server.dart
INFO: 2014-05-17 17:02:38.641: Configured target for / : .helloWorld
INFO: 2014-05-17 17:02:38.654: Setting up VirtualDirectory for /root/dart_apps/web - followLinks: false - jailRoot: true - index files: [index.html]
INFO: 2014-05-17 17:02:38.685: Running on 0.0.0.0:8080
```

DockerのホストのIPアドレスを指定してブラウザから確認します。
 
http://xxx.xxx.xxx:8080

### まとめ
サーバーサイドはGoで書こうと思っていたのですが、Dartも使いやすいマイクロフレームワークもあって、いい感じに書けます。
JavaScriptで書いていたWebアプリはDartで、Goはもっとシステムよりのプログラム向けなのでしょうか。
次回は最近何かと話題の[AngularDart](https://angulardart.org/)を使ってクライアントサイドを見てみます。

