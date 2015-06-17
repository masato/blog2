title: "MeshbluとMongoDBとRedisをDocker Composeで起動する"
date: 2015-04-16 23:00:29
categories:
 - IoT
tags:
 - Meshblu
 - Docker
 - DockerCompose
 - MongoDB
 - Redis
description: これまでMeshbluのリポジトリにあるDockerfileを修正して使っていましたが、最近になりnode 0.10-onbuildをベースイメージにした構成に大きく変更になりました。一つのイメージになっていたMongoDBとRedisもインストールされなくなりました。これにあわせてDocker Copmoseを使ってMeshblu、MongoDB、Redisのコンテナを起動するように変更しようと思います。
---

これまで[Meshblu](https://github.com/octoblu/meshblu)のリポジトリにあるDockerfileを修正して使っていましたが、最近になり[node:0.10-onbuild](https://registry.hub.docker.com/u/library/node/)をベースイメージにした構成に大きく変更になりました。一つのイメージになっていたMongoDBとRedisもインストールされなくなりました。これにあわせて[Docker Copmose](https://docs.docker.com/compose/)を使ってMeshblu、MongoDB、Redisのコンテナを起動するように変更しようと思います。

<!-- more -->

## 準備

以下の作業はすべてrootで行います。Ubuntu 14.04のサーバーを用意して実行します。

### DockerとDocker Composeのインストール

DockerとDocker Composeはそれぞれオフィシャルのone-liner-installerが提供されているのでインストールはとても簡単です。

Dockerを[インストール](http://docs.docker.com/installation/ubuntulinux/)します。

``` bash
$ wget -qO- https://get.docker.com/ | sh
$ docker --version
Docker version 1.6.0, build 4749651
```

Docker Composeを[インストール](https://docs.docker.com/compose/install/)します。

``` bash
$ curl -L https://github.com/docker/compose/releases/download/1.2.0/docker-compose-`uname -s`-`uname -m` > /usr/local/bin/docker-compose; chmod +x /usr/local/bin/docker-compose
$ docker-compose --version
docker-compose 1.2.0
```


### SSL証明書

MeshbluでSSL通信をするため証明書を自己署名で作成します。

``` bash
$ mkdir -p /opt/docker_apps/certs/meshblu
$ cd /opt/docker_apps/certs/meshblu
$ openssl genrsa  -out server.key 4096
$ openssl req -new -batch -key server.key -out server.csr
$ openssl x509 -req -days 3650 -in server.csr -signkey server.key -out server.crt
```

## Docker Composeを使う

Docker Composeは以前Figと呼ばれていたDockerコンテナの管理ツールです。Meshbluのように環境変数の指定が多い場合や、複数コンテナをリンクしてサービスを起動するときオーケストレーションをYAMLに定義して、docker-composeコマンドでまとめて管理できます。Dockerオフィシャルツールなので安心して使えます。

### docker-compose.yml

環境変数`MONGODB_URI`のURLに指定するホスト名をどうやって取得するのか調べていると、[Easily Configure Apps for Multiple Environments with Compose 1.2 and Much More](http://blog.docker.com/2015/04/easily-configure-apps-for-multiple-environments-with-compose-1-2-and-much-more/)に環境変数について書いてありました。

linkで指定した`mongo`の名前はhostsに登録されるためそのまま使えます。

```bash /etc/hosts
172.17.0.2	root_mongo_1 6291efdef64e
172.17.0.2	mongo 6291efdef64e root_mongo_1
172.17.0.2	mongo_1 6291efdef64e root_mongo_1
```

Meshblu、MongoDB、Redisのコンテナをdocker-compose.ymlに定義します。

```yml ~/meshblu_apps/docker-compose.yml
meshblu:
  image: octoblu/meshblu:0c3e5bd
  volumes:
    - .:/var/www
    - ./docker:/etc/supervisor/conf.d
    - /opt/docker_apps/certs/meshblu:/opt/meshblu/certs
  environment:
   - PORT=80
   - MQTT_PORT=1883
   - MQTT_PASSWORD=skynetpass
   - MONGODB_URI=mongodb://mongo:27017/skynet
   - SSL_PORT=443
   - SSL_CERT=/opt/meshblu/certs/server.crt
   - SSL_KEY=/opt/meshblu/certs/server.key
  ports:
    - "80:80"
    - "443:443"
    - "5683:5683"
    - "1883:1883"
  links:
    - redis
    - mongo
redis:
  image: redis
mongo:
  image: mongo
```

### コンテナの起動

指定されたコンテナを3つ起動します。

``` bash
$ docker-compose up -d
Creating root_redis_1...
Creating root_mongo_1...
Creating root_meshblu_1...
```

コンテナをリストします。無事起動しています。複数コンテナをリンクさせて起動するときにDockcer Composeはとても便利に使えます。

``` bash
$ docker-compose ps
            Name                         Command                         State                          Ports
-------------------------------------------------------------------------------------------------------------------------
root_meshblu_1                 npm start                      Up                             0.0.0.0:1883->1883/tcp,
                                                                                             0.0.0.0:443->443/tcp,
                                                                                             0.0.0.0:5683->5683/tcp,
                                                                                             0.0.0.0:80->80/tcp, 9000/tcp
root_mongo_1                   /entrypoint.sh mongod          Up                             27017/tcp
root_redis_1                   /entrypoint.sh redis-server    Up                             6379/tcp
```