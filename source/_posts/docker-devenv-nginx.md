title: 'Dockerで開発環境をつくる - Nginx Part1'
date: 2014-07-28 22:22:56
tags:
 - DockerDevEnv
 - Nginx
description: Dockerで開発環境をつくる - Passenger Nginx と Node.jsDockerのHTTP Routing - Part4 xip.io と Nginx と Node.jsで、LuaやNode.jsをホストするNginxのコンテナは作っていましたが、ふつうに静的コンテンツをホストするNginxが必要になったので用意します。ローカルにある静的コンテンツをコンテナにアタッチできるので、ちょっとしたHTMLファイルをローカルで編集しながらの確認作業に便利に使えます。
---

* `Update 2014-07-29`: [Dockerで開発環境をつくる - Nginx Part2: Hello World, Again](/2014/07/29/docker-devenv-nginx-hello-world-again/)
* `Update 2014-08-04`: [Dockerで開発環境をつくる - Nginx Part3: Supervisorでデモナイズ](/2014/08/04/docker-devenv-nginx-supervisor/)

[Dockerで開発環境をつくる - Passenger Nginx と Node.js](/2014/07/22/docker-devenv-passenger-nodejs/)や[DockerのHTTP Routing - Part4: xip.io と Nginx と Node.js](/2014/07/23/docker-reverse-proxy-xipio-nginx/)で、LuaやNode.jsをホストするNginxのコンテナは作っていましたが、ふつうに静的コンテンツをホストするNginxが必要になったので用意します。

ローカルにある静的コンテンツをコンテナにアタッチできるので、ちょっとしたHTMLファイルをローカルで編集しながらの確認作業に便利に使えます。


<!-- more -->

### Nginxの公式ビルド

[Docker Hub](https://registry.hub.docker.com/)から、Nginxのリポジトリを[検索](https://registry.hub.docker.com/search?q=nginx&s=downloads)します。
[Dockerfile Projext](http://dockerfile.github.io/)のビルドと公式ビルドのDownload数が多いです。

今回は、ボリュームが簡単にアタッチできる[公式ビルド](https://registry.hub.docker.com/_/nginx/)を使用します。

### プロジェクトの作成

適当なディレクトリに静的コンテンツを配置するプロジェクトを作成します。

``` bash
$ sudo mkdir -p /opt/ngix_apps/hello
$ sudo chown masato:masato /opt/ngix_apps/hello
$ cd !$
```

`Hello World`のindex.htmlを作成します。

``` bash
$ echo Hello World > /opt/ngix_apps/hello/index.html
```

### docker run

基本的なrunの書式です。

``` bash
$ docker run --rm -v /some/content:/usr/local/nginx/html:ro nginx
```


先ほど作成した、Dockerホストの静的コンテンツのディレクトリをボリュームにアタッチします。
ポートはテスト用に8084にマップして、disposableにコンテナを起動します。

``` bash
$ docker pull nginx
$ docker run --rm -v /opt/ngix_apps/hello:/usr/local/nginx/html:ro -p 8084:80 nginx
```
 
### 確認

Dockerホストでブラウザを開いて確認します。

{% img center /2014/07/28/docker-devenv-nginx/hello-world.png %}
