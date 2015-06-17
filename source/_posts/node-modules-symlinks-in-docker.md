title: "Dockerのnpm installをnode_modulesのシムリンクとボリュームを使って効率化する"
date: 2015-03-13 21:29:01
tags:
 - Meshblu
 - Mosca
 - Docker
 - IoT
 - nodemodules
 - Nodejs
 - nodegyp
description: Node.jsアプリのDockerイメージを作っているとpackage.jsonのnpm installのときにnode-gypによるネイティブモジュールのビルドが走ることがあります。Node.jsのコードやDockerfileの構成を変更するたびにビルドされて長いときは数分かかります。なにか良い方法がないかググっているとnpm package.json and docker (mounting it…)という記事を見つけました。このソリューションを使っていまビルドに時間がかかっているMeshbluのDockerイメージ作成を修正します。
---

Node.jsアプリのDockerイメージを作っているとpackage.jsonの`npm install`のときに[node-gyp](https://github.com/TooTallNate/node-gyp)によるネイティブモジュールのビルドが走ることがあります。Node.jsのコードやDockerfileの構成を変更するたびにビルドされて長いときは数分かかります。なにか良い方法がないかググっていると[npm package.json and docker (mounting it…)](http://stackoverflow.com/questions/26757264/npm-package-json-and-docker-mounting-it)という記事を見つけました。このソリューションを使っていまビルドに時間がかかっているMeshbluのDockerイメージ作成を修正します。


<!-- more -->

## Meshblu

IoTプラットフォームの[Meshblu](https://github.com/octoblu/meshblu)は内部でMQTTブローカーにMoscaを使い、LebelDBなどのネイティブモジュールのビルドに時間がかかります。またMeshbluのリポジトリのはDockerfileのままではビルドに失敗してしまいます。いくつか修正しながら`node_modules`を効率的にビルドできるようにしていきます。まず`git clone`してデフォルトのディレクトリ状態を確認します。

``` bash
$ cd ~/docker_apps
$ git clone https://github.com/octoblu/meshblu.git
$ cd meshblu
$ tree -L 1
.
├── Dockerfile
├── LICENSE
├── README.md
├── app.json
├── app.yml
├── config.js
├── demo.html
├── docker
├── lib
├── meshblu.sublime-project
├── newrelic.js
├── package.json
├── public
├── server.js
└── test
```

## Dockerイメージのビルド修正

### アプリディレクトリの作成

デフォルトのリポジトリの状態だとDockerfileと同じ階層をそのまま`/var/www`にデプロイする形でイメージをビルドしています。server.jsなどのアプリケーションと、package.jsonをアプリディレクトリに移動します。このディレクトリで`npm install`するので
`node_modules`が配置されます。

``` bash
$ mkdir -p ./var/www
$ mv demo.html lib newrelic.js public server.js test ./var/www
$ tree -L 3
.
├── Dockerfile
├── LICENSE
├── README.md
├── app.json
├── app.yml
├── config.js
├── docker
│   ├── config.js.docker
│   └── supervisor.conf
├── meshblu.sublime-project
├── package.json
└── var
    └── www
        ├── demo.html
        ├── lib
        ├── newrelic.js
        ├── public
        ├── server.js
        └── test
```

### Dockerfileにパッケージ追加

リポジトリからcloneした状態のDockerfileでビルドすると`zmq.h`と`dns_sd.h`がみつからないので失敗します。必要なパッケージをインストールします。

``` bash ~/docker_apps/meshblu/Dockerfile
RUN apt-get install -y libzmq-dev libavahi-compat-libdnssd-dev
```

### Dockerfileのディレクトリ修正

`node_modules`ディレクトリやSupervisor設定ファイルの管理を別にして修正後にDockerイメージを再ビルドしなくても`docker restart`できるようにします。アプリや設定をnpmやインフラのパッケージのインストールを分離することができます。

* `node_modules`をアプリディレクトリの外の`/dist/node_modules`に作成して、アプリディレクトリにはシムリンクを作る
* Dockerfileの最後にCOPYコマンドで、アプリケーションを変更してもnpmインストールが起きないようにする
* Supervisorの設定ファイルはボリュームからアタッチするように変更する

``` bash ~/docker_apps/meshblu/Dockerfile
...
#ADD . /var/www
#RUN cd /var/www && npm install
COPY ./var/www/package.json /var/www/
RUN mkdir -p /dist/node_modules && \
  ln -s /dist/node_modules /var/www/node_modules && \
  cd /var/www && npm install
#ADD ./docker/config.js.docker /var/www/config.js
#ADD ./docker/supervisor.conf /etc/supervisor/conf.d/supervisor.conf
...
CMD ["/usr/bin/supervisord", "-n", "-c", "/etc/supervisor/supervisord.conf"]
VOLUME /etc/supervisor/conf.d
COPY ./var/www/ /var/www
```

Meshbluの設定ファイルはDockerfile内でコメントアウトしたので、アプリディレクトリに移動します。

``` bash
$ mv ./docker/config.js.docker ./var/www/config.js
```

### アプリディレクトリのボリューム

最後に`/var/www`のアプリディレクトリもDockerイメージから分離してボリュームでアタッチできるようにします。Dockerホストには`/dist/node_modules`ディレクトリは存在しませんが、コンテナ内では存在するシムリンクを作成します。

``` bash
$ ln -s /dist/node_modules ./var/www/node_modules
```

### 修正後のディレクトリ

Meshbluのリポジトリは以下のように修正しました。

``` bash
$ tree -L 3
.
├── Dockerfile
├── LICENSE
├── README.md
├── app.json
├── app.yml
├── config.js
├── docker
│   └── supervisor.conf
├── meshblu.sublime-project
└── var
    └── www
        ├── config.js
        ├── demo.html
        ├── lib
        ├── newrelic.js
        ├── node_modules -> /dist/node_modules
        ├── package.json
        ├── public
        ├── server.js
        └── test
```

## Dockerイメージのビルドと実行

Dockerイメージをビルドしてコンテナの起動を確認します。

``` bash
$ docker build -t meshblu .
$ docker run -d --name meshblu \
  -p 3000 \
  -v $PWD/var/www:/var/www \
  -v $PWD/docker:/etc/supervisor/conf.d \
  meshblu
```
