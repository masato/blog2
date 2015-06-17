title: 'Dockerで開発環境をつくる - Passenger Nginx と Node.js'
date: 2014-07-22 23:00:29
tags:
 - DockerDevEnv
 - Nginx
 - Nodejs
 - Passenger
description: Nginxの手軽なDockerコンテナが必要になったので、phusion/baseimageでDockerイメージを作ろうと思い、Node.jsとNginxのサンプルを探していました。Docker Misconceptionsのポストでpassenger-dockerを紹介していました。PassengerはRubyだけかと思っていましたが、Python,Node.js,Meteorまで対応しています。Node.jsのpm2やforeverをDockerでどうやって使うのか考えるよりPassengerで簡単にできそうです。Node.js用のイメージはphusion/baseimageから派生したphusion/passenger-nodejsになります。
---

* `Update 2014-07-28`: [Dockerで開発環境をつくる - Nginx](/2014/07/28/docker-devenv-nginx/)

Nginxの手軽なDockerコンテナが必要になったので、`phusion/baseimage`でDockerイメージを作ろうと思い、Node.jsとNginxのサンプルを探していました。[Docker Misconceptions](https://devopsu.com/blog/docker-misconceptions/)のポストで[passenger-docker](https://github.com/phusion/passenger-docker)を紹介していました。

PassengerはRubyだけかと思っていましたが、Python,Node.js,Meteorまで対応しています。
Node.jsのpm2やforeverをDockerでどうやって使うのか考えるよりPassengerで簡単にできそうです。

Node.js用のイメージは`phusion/baseimage`から派生した`phusion/passenger-nodejs`になります。

<!-- more -->

### Dockerfile

最初に今回作成したDockerfileです。

``` bash ~/docker_apps/nodejs/Dockerfile
FROM phusion/passenger-nodejs:0.9.11

# Set correct environment variables.
ENV HOME /root

# Use baseimage-docker's init process.
CMD ["/sbin/my_init"]

# Node.js and Meteor support.
RUN /build/nodejs.sh

# ...put your own build instructions here...

# Regenerate SSH host keys.
RUN /etc/my_init.d/00_regen_ssh_host_keys.sh

# Install an SSH of your choice.
ADD my_key.pub /tmp/your_key
RUN cat /tmp/your_key >> /root/.ssh/authorized_keys && rm -f /tmp/your_key

# Using Nginx and Passenger
RUN rm -f /etc/service/nginx/down

# ...commands to place your web app in /home/app/webapp...
RUN rm -rf /etc/nginx/sites-enabled/default
ADD webapp.conf /etc/nginx/sites-enabled/
RUN mkdir -p /home/app/webapp/public
ADD app.js /home/app/webapp/

# Clean up APT when done.
RUN apt-get clean && rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*
```

### Node.jsアプリ

[Phusion Passenger: Node.js tutorial](https://github.com/phusion/passenger/wiki/Phusion-Passenger%3A-Node.js-tutorial)を読むとNode.jsをPassengerで動かす場合、nodeコマンドで起動するファイルはapp.jsである必要があります。

``` bash
application directory
  |
  +-- app.js
  |
  +-- public/
  |
  +-- tmp/
```

単純な`Hellow World`を作成します。
listenするポートは8000にしていますが、Passengerの場合意味がなく、3000ポートでlistenするようです。

``` node ~/docker_apps/nodejs/app.js
var http = require('http');
http.createServer(function (request, response) {
  response.writeHead(200, {'Content-Type': 'text/plain'});
  response.end('Hello World\n');
}).listen(8000);
```

### Nginxを有効にする

`phusion/passenger-nodejs`イメージではでフォルトでNginxが有効になっていません。
downファイルを削除して有効にします。

``` bash ~/docker_apps/nodejs/Dockerfile
RUN rm -f /etc/service/nginx/down
```

### site-enabled

Passengerで有効にするWebアプリの設定をconfに書いて配置します。


``` bash ~/docker_apps/nodejs/webapp.conf
server {
    listen 80;
    server_name localhost;
    root /home/app/webapp/public;
    passenger_enabled on;
    passenger_user app;
}
```

Dockerfileでは以下のように、最初にdefaultを削除します。defaultでは80でlistenしているため、
追加するconfと競合してしまいます。

``` bash ~/docker_apps/nodejs/Dockerfile
ADD webapp.conf /etc/nginx/sites-enabled/webapp.conf
RUN mkdir /home/app/webapp
ADD app.js /home/app/webapp
```

### build と run

Dockerイメージをビルドします。

``` bash
$ cd ~/docker_apps/nodejs
$ docker build -t masato/nodejs .
```

Dockerコンテナを実行します。

``` bash
$ docker run --rm -p 80:80 -i -t masato/nodejs
```

Dockerホストからブラウザで確認します。

{% img center /2014/07/22/docker-devenv-passenger-nodejs/nodejs_hello_world.png %}

### docker-bashコマンド

[passenger-docker](https://github.com/phusion/passenger-docker)にはDockerホストにインストールして使う、便利なコマンドがあります。

``` bash
$ cd Downloads
$ curl --fail -L -O https://github.com/phusion/baseimage-docker/archive/master.tar.gz && \
   tar xzf master.tar.gz && \
   sudo ./baseimage-docker-master/install-tools.sh
...
+ cp tools/docker-bash /usr/local/bin/
+ cp tools/docker-ssh /usr/local/bin/
+ cp tools/baseimage-docker-nsenter /usr/local/bin/
+ mkdir -p /usr/local/share/baseimage-docker
+ cp image/insecure_key /usr/local/share/baseimage-docker/
+ chmod 644 /usr/local/share/baseimage-docker/insecure_key
```

書式です。

```
$ docker-bash YOUR-CONTAINER-ID
```

コンテナIDを指定して、SSHログインできます。

``` bash
$ sudo docker-bash 8bbcde370741
root@8bbcde370741:/#
```

また、コンテナにSSHログインして、ワンショットのコマンドを実行できます。

```
$ docker-bash 8bbcde370741 echo hello world
```

### まとめ

PassengerでNode.jsを初めて使ってみましたが、通常の利用では自動的にNginxがリバースプロキシしてくれるのはかなり便利です。
ただ、複数のDockerコンテナが開いているポートへ、Nginxがリバースプロキシしてほしかったので、ちょっと用途が違いました。

仕方がないので自分で`phusion/baseimage`のrunitを使いイメージを作ることにします。
