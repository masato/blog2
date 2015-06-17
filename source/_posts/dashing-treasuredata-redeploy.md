title: 'DashingとTreasure Data - Part6: docker-registryのバージョン'
date: 2014-09-03 01:21:04
tags:
 - docker-registry
 - IDCFオブジェクトストレージ
 - TreasureData
 - Dashing
 - LinuxMint17
description: Dockerイメージはオブジェクトストレージ保存していたので、docker-registryとDashingコンテナをデプロイしていたCoreOSを破棄していました。必要なときに別のDockerホストで`docker pull`すれば良いと思っていたのですが、使いたいときにイメージが取得できない状況になり困りました。最終的には動いたのですが、今後のバージョンアップを考えると、docker-registryでのイメージ保存の仕様はしっかり理解する必要があります。
---

Dockerイメージはオブジェクトストレージ保存していたので、docker-registryとDashingコンテナをデプロイしていたCoreOSを破棄していました。必要なときに別のDockerホストで`docker pull`すれば良いと思っていたのですが、使いたいときにイメージが取得できない状況になり困りました。最終的には動いたのですが、今後のバージョンアップを考えると、docker-registryでのイメージ保存の仕様はしっかり理解する必要があります。

<!-- more -->

### docker-registryの再デプロイ

LinuxMint17の開発環境Dockerホストに、docker-registryをデプロイします。
設定ファイルのconfig.ymlを作成します。`dev flavor`のloglevelが必須になっています。今回嵌まったのも設定ファイルのリファクタリングの影響のようです。

boto経由でIDCFオブジェクトストレージをイメージ保存のバックエンドに使います。
キー情報は環境変数から取得できるようにします。

``` yaml ~/registry_conf/config.yml
common:
    loglevel: info
    secret_key: _env:REGISTRY_SECRET
    standalone: true
    disable_token_auth: true

dev:
    loglevel: debug
    storage: s3
    s3_access_key: _env:AWS_S3_ACCESS_KEY
    s3_secret_key: _env:AWS_S3_SECRET_KEY
    s3_bucket: docker-registry
    boto_bucket: docker-registry
    boto_host: ds.jp-east.idcfcloud.com
    s3_encrypt: false
    s3_secure: true
    storage_path: /images
```

### docker-registry v0.8.1は失敗

2014-08-29現在で、docker-registryのlatestはv0.8.1です。

``` bash
$ curl localhost:5000
"docker-registry server (dev) (v0.8.1)
```

v0.8から設定ファイルの書式やAPIに`breaking changes`が入っています。

* [reparing 0.8](https://github.com/docker/docker-registry/pull/514)
* [Registry 0.8 release and API breaking changes (config)](https://github.com/bacongobbler/docker-registry-driver-swift/issues/12)
* [Config rehaul #444](https://github.com/docker/docker-registry/pull/444)

v0.7.2には、STANDALONE設定問題があったりと、ここ最近はバージョンに注意が必要です。
v0.6.9のときにpushしたイメージを、v0.8.1からpullすると405が返りエラーになりました。 

``` bash
$ docker pull localhost:5000/dashing
Pulling repository localhost:5000/dashing
2014/08/29 12:59:39 HTTP code: 405
```

### docker-registry v0.6.9で成功

開発環境Dockerホストに、以前動いていたv0.6.9を指定してregistryをpullします。

``` bash
$ docker pull registry:0.6.9
```

`REGISTRY_SECRET`はランダム生成するようにしました。

``` bash
$ docker run -d -p 5000:5000 -v /home/mshimizu/registry_conf:/registry_conf -e DOCKER_REGISTRY_CONFIG=/registry_conf/config.yml -e AWS_S3_ACCESS_KEY="{access_key}" -e AWS_S3_SECRET_KEY="{secret_key}" -e REGISTRY_SECRET=`openssl rand -base64 64 | tr -d '\n'` registry:0.6.9
```

### Dashingコンテナの起動

本番環境Dockerホストから、開発環境Dockerホストのdocker-registryへ接続確認します。

``` bash
$ curl 10.1.1.32:5000
"docker-registry server (dev) (v0.6.9)
```

docker-registryからイメージをpullします。

``` bash
$ docker pull 10.1.1.32:5000/dashing
```

`TD_API_KEY`の環境変数に、TreasureDataのapikeyを指定して、コンテナを起動します。

``` bash
$ docker run -d -p 80 --name dashing -e TD_API_KEY={td_api_key} 10.1.1.32:5000/dashing /sbin/my_init
```

### HTTP Routing

psでコンテナの状態を確認します。

``` bash
$ docker ps
CONTAINER ID        IMAGE                                   COMMAND              CREATED             STATUS              PORTS                                                NAMES
e693c30073b4        10.1.1.32:5000/dashing:latest           /sbin/my_init        3 seconds ago       Up 2 seconds        0.0.0.0:49154->80/tcp                                dashing
```

起動したDashingコンテナのIPアドレスを確認します。 

``` bash
$ getip e693c30073b4
172.17.0.22
```

OpenResty+Lua+Redisで、外から接続できるように`HTTP Routing`の設定を登録します。

``` bash
$ docker run -it --rm --link openresty:redis dockerfile/redis bash -c 'redis-cli -h $REDIS_PORT_6379_TCP_ADDR set dashing.10.1.2.244.xip.io 172.17.0.22:80'
```

### 確認

ブラウザから確認します。

https://dashing.10.1.2.244.xip.io/cloud

### まとめ

無事に新しいDockerホストにもTreasureDataとDashingコンテナが動くようになりました。
データベースはTreasureDataのクラウドサービスを使っているので、コンテナはステートレスに使えます。

Dockerでステートフルなコンテナは、なるべくクラウドサービスを利用したいです。
