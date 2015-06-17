title: "Node-RED on Docker - Part3: docker buildエラー修正とnodesDirの設定"
date: 2015-02-13 20:01:00
tags:
 - Docker
 - Node-RED
 - Nodejs
description: google/nodejs-runtimeをbaseイメージにして作成したNode-REDのDockerイメージは、ログを見るとbuildでエラーが発生していました。node-icu-charset-detectorモジュールのnode-gyp rebuildに失敗しています。このままでも動作はしていますがエラーが出ないようにDockerイメージを作り直します。
---

[google/nodejs-runtime](https://registry.hub.docker.com/u/google/nodejs-runtime/)をbaseイメージにして作成したNode-REDのDockerイメージは、ログを見るとbuildでエラーが発生していました。node-icu-charset-detectorモジュールの`node-gyp rebuild`に失敗しています。このままでも動作はしていますがエラーが出ないようにDockerイメージを作り直します。

<!-- more -->

## docker buildのエラー

node-gypを使ったネイティブモジュールのビルドでエラーが出ていました。

``` bash
$ docker build -t node-red .
...
> node-icu-charset-detector@0.0.7 install /app/node_modules/irc/node_modules/node-icu-charset-detector
> node-gyp rebuild

make: Entering directory `/app/node_modules/irc/node_modules/node-icu-charset-detector/build'
  CXX(target) Release/obj.target/node-icu-charset-detector/node-icu-charset-detector.o
../node-icu-charset-detector.cpp:5:28: fatal error: unicode/ucsdet.h: No such file or directory
compilation terminated.
make: *** [Release/obj.target/node-icu-charset-detector/node-icu-charset-detector.o] Error 1
make: Leaving directory `/app/node_modules/irc/node_modules/node-icu-charset-detector/build'
gyp ERR! build error
gyp ERR! stack Error: `make` failed with exit code: 2
gyp ERR! stack     at ChildProcess.onExit (/nodejs/lib/node_modules/npm/node_modules/node-gyp/lib/bu
ild.js:267:23)
gyp ERR! stack     at ChildProcess.emit (events.js:98:17)
gyp ERR! stack     at Process.ChildProcess._handle.onexit (child_process.js:810:12)
gyp ERR! System Linux 3.13.0-39-generic
gyp ERR! command "node" "/nodejs/lib/node_modules/npm/node_modules/node-gyp/bin/node-gyp.js" "rebuild"
gyp ERR! cwd /app/node_modules/irc/node_modules/node-icu-charset-detector
gyp ERR! node -v v0.10.33
gyp ERR! node-gyp -v v1.0.1
gyp ERR! not ok
npm WARN optional dep failed, continuing node-icu-charset-detector@0.0.7
```

node-icu-charset-detectorのビルドで`unicode/ucsdet.h`が見つからないようです。

``` bash
../node-icu-charset-detector.cpp:5:28: fatal error: unicode/ucsdet.h: No such file or directory
```

### apt-fileを使ってパッケージを検索

apt-fileコマンドでunicode/ucsdet.hを検索します。必要なヘッダは`libicu-dev`パッケージをインストールすると良いみたいです。
 
``` bash
$ sudo apt-get update
$ sudo apt-get install apt-file
$ sudo apt-file update
$ apt-file search unicode/ucsdet.h
libicu-dev: /usr/include/x86_64-linux-gnu/unicode/ucsdet.h
```

### google/nodejs-runtime

現在のDockerfileは[google/nodejs-runtime](https://registry.hub.docker.com/u/google/nodejs-runtime/)をbaseイメージに指定しているだけです。

``` bash
$ echo FROM google/nodejs-runtime > Dockerfile
```

`libicu-dev`をインストールしたいのですがONBUILDで`npm install`を実行しているので、派生させたDockerfileでは`npm install`の実行後になってしまい`node-gyp rebuild`に間に合いません。

``` bash Dockerfile
FROM google/nodejs

WORKDIR /app
ONBUILD ADD package.json /app/
ONBUILD RUN npm install
ONBUILD ADD . /app

EXPOSE 8080
CMD []
ENTRYPOINT ["/nodejs/bin/npm", "start"]
```

[google/nodejs](https://registry.hub.docker.com/u/google/nodejs/)をbaseイメージにしてDockerfileを作り直すことにします。

## docker buildのやり直し

Node-REDをgit cloneしたディレクトリに移動します。

``` bash
$ cd ~/docker_apps/nod-red
```

settings.jsを編集して、追加のnode用にjsとhtmlファイルを配置するディレクトリを指定します。

``` js ~/docker_apps/nod-red/settings.js
...
nodesDir: '/data/nodes',
...
```

nodesDirに指定したディレクトリはコンテナの起動時にDockerホストの`/opt/nodes`にマップします。

``` bash
$ mkdir -p /opt/nodes
```

ビルドのテスト用にres.jsに-vフラグを追加してデバッグします。

``` js  ~/docker_apps/nod-red/package.js
...
    "scripts"      : {
        "start": "node red.js -v",
        "test": "./node_modules/.bin/grunt"
    },
...
```

### google/nodejs

[google/nodejs](https://registry.hub.docker.com/u/google/nodejs/)をbaseイメージにしたDockerfileです。

``` bash ~/docker_apps/nod-red/Dockerfile
FROM google/nodejs

RUN apt-get update && \
  apt-get install -y libicu-dev

WORKDIR /app
ADD package.json /app/
RUN npm install
ADD . /app

EXPOSE 1880
CMD []
ENTRYPOINT ["/nodejs/bin/npm", "start"]

RUN apt-get clean && rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*
```

Dockerイメージをビルドします。今度はnode-icu-charset-detectorの`node-gyp rebuild`に成功しました。

``` bash
$ cd docker_apps/nod-red
$ docker build -t node-red .
...
> node-icu-charset-detector@0.0.7 install /app/node_modules/irc/node_modules/node-icu-charset-detect
or
> node-gyp rebuild

make: Entering directory `/app/node_modules/irc/node_modules/node-icu-charset-detector/build'
  CXX(target) Release/obj.target/node-icu-charset-detector/node-icu-charset-detector.o
  SOLINK_MODULE(target) Release/obj.target/node-icu-charset-detector.node
  SOLINK_MODULE(target) Release/obj.target/node-icu-charset-detector.node: Finished
  COPY Release/node-icu-charset-detector.node
make: Leaving directory `/app/node_modules/irc/node_modules/node-icu-charset-detector/build'
...
irc@0.3.9 node_modules/irc
├── ansi-color@0.2.1
├── irc-colors@1.1.0 (hashish@0.0.4)
├── node-icu-charset-detector@0.0.7
└── iconv@2.1.5 (nan@1.4.3)
...
```

テスト用に使い捨てのコンテナを起動してデバッグを確認します。

``` bash
$ docker run --rm \
  --name node-red \
  -p 1880:1880 \
  -v /opt/nodes:/data/nodes \
  node-red 
> node-red@0.9.1 start /app
> node red.js -v


Welcome to Node-RED
===================

13 Feb 09:04:53 - [info] Version: 0.9.1.git
13 Feb 09:04:53 - [info] Loading palette nodes
13 Feb 09:04:54 - [warn] ------------------------------------------
13 Feb 09:04:54 - [warn] [arduino] Error: Cannot find module 'arduino-firmata'
13 Feb 09:04:54 - [warn] [rpi-gpio] Info : Ignoring Raspberry Pi specific node.
13 Feb 09:04:54 - [warn] [redisout] Error: Cannot find module 'redis'
13 Feb 09:04:54 - [warn] [mongodb] Error: Cannot find module 'mongodb'
13 Feb 09:04:54 - [warn] ------------------------------------------
13 Feb 09:04:54 - [info] Server now running at http://127.0.0.1:1880/
13 Feb 09:04:54 - [info] Flows file not found : flows_e816455bc0d9.json
13 Feb 09:04:54 - [info] Starting flows
```

ブラウザで動作確認をします。

http://10.1.3.67:1880



