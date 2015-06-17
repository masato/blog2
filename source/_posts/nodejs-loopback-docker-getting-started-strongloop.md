title: "Node.js LoopBack on Docker - Part2: StrongLoopをインストール"
date: 2014-12-25 22:14:46
tags:
 - MBaaS
 - LoopBack
 - StrongLoop
 - Nodejs
 - nvm
 - Isomorphic
 - DockerDevEnv
 - ngrok
description: さっそくStrongLoopを使ってみるために簡単なNode.jsのDockerイメージを作成しました。LoopBackはMBaaSの進化としてSingle Page Appsのためのデータベースと一体になったAPI Serverとして使うと便利です。Parseだと任意のnpmモジュールが使えなかったり、ビジネスロジックの記述に制限があります。フロントエンドをAngularJSで書くとIsomorphicなアプリができるのも魅力的です。
---

さっそくStrongLoopを使ってみるために簡単なNode.jsの[Dockerイメージ](https://registry.hub.docker.com/u/masato/strongloop/)を作成しました。LoopBackはMBaaSの進化としてSingle Page Appsのためのデータベースと一体になったAPI Serverとして使うと便利です。Parseだと任意のnpmモジュールが使えなかったり、ビジネスロジックの記述に制限があります。フロントエンドをAngularJSで書くとIsomorphicなアプリができるのも魅力的です。


<!-- more -->

### masato/strongloop

LoopBackを使うためにNode.jsをインストールしたDockerイメージを作成しました。[masato/strongloop](https://registry.hub.docker.com/u/masato/strongloop/)のDockerfileです。

``` bash ~/docker_apps/strongloop/Dockerfile
FROM ubuntu:14.04.1
MAINTAINER Masato Shimizu <ma6ato@gmail.com>
ENV NODE_VERSION v0.10
ENV HOME /root
RUN apt-get update && \
    DEBIAN_FRONTEND=noninteractive \
    apt-get install -y curl git 
RUN git clone git://github.com/creationix/nvm.git ${HOME}/.nvm && \
    /bin/bash -c 'source ${HOME}/.nvm/nvm.sh && \
                  nvm install ${NODE_VERSION} && \
                  nvm use ${NODE_VERSION} && \
                  nvm alias default ${NODE_VERSION}' && \
    echo '[[ -s "${HOME}/.nvm/nvm.sh" ]] && source "${HOME}/.nvm/nvm.sh"' >> ${HOME}/.profile && \
    /bin/bash -c 'source ${HOME}/.profile && \
                  npm install -g npm && \
                  npm install -g strongloop && \
                  npm cache clear'
EXPOSE 3000 53322
RUN apt-get clean && rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*
```

Docker開発環境を起動します。npmをrootで作業をすると警告がでるので、作業ユーザーにスイッチします。

``` bash
$ docker run -it --name loopback --rm masato/strongloop /bin/bash
$ su - docker
```

### slcコマンドを使う

[Getting Started is Easy](http://strongloop.com/get-started/)を読みながらslcコマンドでプロジェクトを作成します。Yeomanのラッパーになっています。


``` bash
$ slc loopback

     _-----_
    |       |    .--------------------------.
    |--(o)--|    |  Let's create a LoopBack |
   `---------´   |       application!       |
    ( _´U`_ )    '--------------------------'
    /___A___\
     |  ~  |
   __'.___.'__
 ´   `  |° ´ Y `

? What's the name of your application? spike
? Enter name of the directory to contain the project: spike
   create spike/
     info change the working directory to spike
...
Next steps:

  Change directory to your app
    $ cd spike

  Create a model in your app
    $ slc loopback:model

  Optional: Enable StrongOps monitoring
    $ slc strongops

  Run the app
    $ slc run .
```

画面にガイドが表示されるので、次にモデルをつくります。Bookモデルの作成をします。あとでMongoDBを使いますが最初にin-memoryデータベースで確認します。適当にプロパティを追加します。モデルは[PersistedModel](http://docs.strongloop.com/display/public/LB/PersistedModel+REST+API)を選択するとCRUDの操作とREST APIが自動的に作成されて公開できます。

``` bash
$ slc loopback:model Book
? Enter the model name: Book
? Select the data-source to attach Book to: db (memory)
? Select model's base class: PersistedModel
? Expose Book via the REST API? Yes
? Custom plural form (used to build REST URL):
Let's add some Book properties now.

Enter an empty property name when done.
? Property name: name
   invoke   loopback:property
? Property type: string
? Required? Yes

Let's add another Book property.
Enter an empty property name when done.
? Property name:
$
```


### ngrokでREST APIを公開する

`slc run`コマンドを実行してREST APIを公開します。

``` run
$ slc run
INFO strong-agent API key not found, StrongOps dashboard reporting disabled.
Generate configuration with:
    npm install -g strongloop
    slc strongops
See http://docs.strongloop.com/strong-agent for more information.
supervisor running without clustering (unsupervised)
Browse your REST API at http://0.0.0.0:3000/explorer
Web server listening at: http://0.0.0.0:3000/
```

`docker inspect`で開発用コンテナのIPアドレスを確認します。

``` bash
$ docker inspect --format="&#123;&#123; .NetworkSettings.IPAddress }}" loopback
172.17.1.154
```

別のシェルを開き、ngrokを起動して公開用のURLを取得します。

``` bash
$ docker run -it --rm wizardapps/ngrok:latest ngrok 172.17.1.154:3000
```

ngrokのURLからStrongLoop API Explorerを開きます。Usersのモデルははデフォルトで作成されます。BooksのREST APIが作成されました。

https://56110a8c.ngrok.com/explorer/

{% img center /2014/12/25/nodejs-loopback-docker-getting-started-strongloop/strongloop_api_explorer.png %}

[Swagger 2.0](http://swagger.io/)の仕様に準拠したUIを使うことができます。

{% img center /2014/12/25/nodejs-loopback-docker-getting-started-strongloop/strongloop_api_explorer_ui.png %}
