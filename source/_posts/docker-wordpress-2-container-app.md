title: "DockerにWordPress4.0日本語版を2-Container-Appインストール"
date: 2014-09-08 00:40:07
tags:
 - Tutum
 - Docker
 - WordPress
 - MySQL
description: Salt+DockerでWordPressをインストールしたのですが、WordPress4.0日本語版にしてもう少し実践的にコンテナをつくります。TutumのHow To Build A 2-Container App with Docker and Tutumを参考にして、WordPressとMySQLの2つのコンテナを作成してLinkさせます。 
---

* `Update 2014-10-01`: [DockerのWordPressコンテナをcommitしてもvolumeがimageに含まれなくて困った](/2014/10/01/docker-wordpress-duplicate-volumes/)


[Salt+DockerでWordPressをインストール](/2014/09/07/salt-idcf-docker-wordpress/)したのですが、WordPress4.0日本語版にしてもう少し実践的にコンテナをつくります。

Tutumの[How To Build A 2-Container App with Docker and Tutum](http://blog.tutum.co/2014/02/06/how-to-build-a-2-container-app-with-docker-and-tutum/)を参考にして、WordPressとMySQLの2つのコンテナを作成してLinkさせます。

<!-- more -->


### WordPress4.0から言語選択ができる

[tutum-docker-wordpress-nosql](https://github.com/tutumcloud/tutum-docker-wordpress-nosql)を`git clone`してDockerfileをWordPressを日本語版に変更しようと思ったのですが、[WordPress4.0以降は「define(‘WPLANG’, ‘ja‘);」言語指定が不要になった](http://algorhythnn.jp/blg/2014/09/06/wp4-wplang-ja-useless/)という記事を見つけました。
WordPress4.0からのサイト言語選択ができるようになっているそうです。
ただし[WP Multibyte Patch](http://wordpress.org/plugins/wp-multibyte-patch/)は別途プラグインとしてインストールする必要があります。

そもそも日本語版というものが存在することが疑問だったので、ようやく普通に国際版が使えるようになりました。

### TutumのWordPressは3.9.2

nsenterでWordPressコンテナに接続します。

``` bash
$ nse d7e936d5bceb
```

バージョン情報を確認します。

``` php  /app/wp-includes/version.php
$wp_version = '3.9.2';
```

残念ながらregistryからpullできるTutumのイメージは残念ながらまだ4.0にはなっていませんでした。

### WordPressイメージをビルドする

最初に戻って作業マシンで`git clone`してDockerイメージをビルドします。

``` bash
$ cd ~/docker_apps
$ git clone https://github.com/tutumcloud/tutum-docker-wordpress-nosql.git
```

WordPress4.0日本語版をダウンロードして、`/app`ディレクトリに展開します。
パッケージのダウンロードサイトを日本に変更して、wgetをインストールします。
それ以外は、`git clone`したままのDockerfileです。

``` bash ~/docker_apps/tutum-docker-wordpress-nosql/Dockerfile
FROM tutum/apache-php:latest
MAINTAINER Masato Shimizu <ma6ato@gmail.com>

# Install packages
RUN sed -i~ -e 's/archive.ubuntu.com/ftp.jaist.ac.jp/' /etc/apt/sources.list \
  && apt-get update \  
  && apt-get -yq install mysql-client wget \
  && rm -rf /var/lib/apt/lists/*

# Download latest version of Wordpress into /app
RUN rm -fr /app && mkdir -p /app \
  && wget http://ja.wordpress.org/wordpress-4.0-ja.tar.gz -P /tmp \
  && tar zxvf /tmp/wordpress-4.0-ja.tar.gz -C /app --strip-components=1

ADD wp-config.php /app/wp-config.php
RUN chown www-data:www-data /app -R

# Add script to create 'wordpress' DB
ADD run-wordpress.sh /run-wordpress.sh
RUN chmod 755 /*.sh

# Modify permissions to allow plugin upload
RUN chmod -R 777 /app/wp-content

# Expose environment variables
ENV DB_HOST **LinkMe**
ENV DB_PORT **LinkMe**
ENV DB_NAME wordpress
ENV DB_USER admin
ENV DB_PASS **ChangeMe**

EXPOSE 80
VOLUME ["/app/wp-content"]
CMD ["/run-wordpress.sh"]
```

Dockerイメージをビルドしてdocker-registryにpushします。

``` bash
$ docker build -t masato/wordpress .
$ docker tag masato/wordpress localhost:5000/wordpress
$ docker push localhost:5000/wordpress
```

デプロイ先のDockerホストでdocker-registryからイメージをpullします。

``` bash
$ docker pull 10.1.1.32:5000/wordpress
$ docker pull tutum/mysql:5.5
```

最初にMySQLコンテナの起動して、WordPressコンテナとリンクさせます。

``` bash
$ docker run -d -e MYSQL_PASS="password" --name db -p 3306:3306 tutum/mysql:5.5
$ docker run -d --link db:db -e DB_PASS="password" -p 80:80 10.1.1.32:5000/wordpress
```

### 確認

ブラウザから確認します。

http://10.1.2.97

インストール画面から日本語表示されるようになりました。

{% img center /2014/09/08/docker-wordpress-2-container-app/wordpress-jp.png %}


