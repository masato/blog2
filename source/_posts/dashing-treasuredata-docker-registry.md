title: 'DashingとTreasure Data - Part4: Docker Registryへ登録'
date: 2014-06-06 01:21:04
tags:
 - Dashing
 - AnalyticSandbox
 - TreasureData
 - IDCFクラウド
 - IDCFオブジェクトストレージ
 - Docker
 - docker-registry
description: IDCFクラウドをDocker開発環境として、Treasure DataのサンプルアプリをPart1,Part2,Part3まで、Dashingのダッシュボードを作成するところまで開発しました。次はプロダクションを想定してイメージを作成します。Gitリポジトリから`git pull`するようにしてなるべく実際のデプロイに近づけています。プロダクションイメージは、後で、開発用のDockerホストとは別の、CoreOSのDockerホストへデプロイするために、以前IDCFオブジェクトストレージに構築したローカルのdocker-registryへ登録します。
---

IDCFクラウドをDocker開発環境として、`Treasure Data`のサンプルアプリを[Part1](/2014/05/27/dashing-treasuredata-install/),[Part2](/2014/05/30/dashing-treasuredata-td/),[Part3](/2014/05/31/dashing-treasuredata-td-job/)まで、Dashingのダッシュボードを作成するところまで開発しました。
次はプロダクションを想定してイメージを作成します。Gitリポジトリから`git pull`してなるべく実際のデプロイに近づけています。

プロダクションイメージは、後で、開発用のDockerホストとは別の、CoreOSのDockerホストへデプロイするために、以前IDCFオブジェクトストレージに構築したローカルの[docker-registry](/2014/05/22/idcf-storage-docker-registry/)へ登録します。

<!-- more -->

### 自分のbaseimageを作成

[Part3](/2014/05/31/dashing-treasuredata-td-job/)で作成した、[phusion/baseimage]がFROMのDockerfileから、自分のbaseimageを作成します。タグでバージョンを管理します。

``` bash
$ cd ~/docker_apps/dashing/phusion
$ docker build -no-cache -t masato/baseimage:1.0 .
```

### プロダクション用イメージの作成

プロダクション用イメージを作成するため、Dockerfileを書きます。
 
まずプロジェクトを作成します。
``` bash
$ mkdir -p ~/docker_apps/dashing
$ cd !$
```

ソースコードをGitからSSHでpullするために、イメージに秘密鍵をコピーします。
この手順が適切なのか疑問ですが、とりあえず`git clone`した後は削除するようにします。

秘密鍵をプロジェクトのディレクトリにコピーします。

``` bash
$ cp ~/.ssh/private_key
```

作成したDockerfileは次のようになります。
``` bash ~/docker_apps/dashing/Dockerfile
FROM masato/baseimage:1.0

RUN mkdir -p /root/.ssh
ADD private_key /root/.ssh/id_rsa
RUN chmod 700 /root/.ssh/id_rsa
RUN echo "Host {Gitリポジトリ}\n\tStrictHostKeyChecking no\n" >> /root/.ssh/config

RUN git clone ssh://{Gitリポジトリ}/var/git/repos/dashing_apps/cloud_td.git /root/cloud_td
RUN rm /root/.ssh/id_rsa
RUN rm /root/.ssh/config

RUN cd /root/cloud_td && bundle install

RUN mkdir /etc/service/thin
ADD thin.sh /etc/service/thin/run
RUN chmod u+x /etc/service/thin/run
```

プロダクション用のイメージを作成します。
``` bash
$ docker build -t masato/dashing:1.0.1 .
```

コンテナをイメージの起動して確認します。
``` bash
$ cd ~/docker_apps/dashing
$ docker run --rm -p 80:80 --name dashing -e TD_API_KEY=xxx masato/dashing:1.0.1 /sbin/my_init
```

### docker-registryへイメージの登録

ローカルの[docker-registry](/2014/05/22/idcf-storage-docker-registry/)が起動していることを確認します。

``` bash
$ curl localhost:5000
"docker-registry server (dev) (v0.7.0)"
```

プロダクション用イメージにタグをつけて、RiakCSをバックエンドにしているプライベートレジストリにpushします。
`private registry`の場合、<username>/<repo_name>の<username>のところが、localhost:5000になります。

``` bash
$ docker tag masato/dashing:1.0.1 localhost:5000/dashing
$ docker push localhost:5000/dashing
```

RiakCSにイメージが保存されていることを、s3cmdを使って確認します。

``` bash
$ s3cmd -c ~/.s3cfg.idcf ls s3://docker-registry/images/repositories/library/dashing/
2014-06-06 04:25      4180   s3://docker-registry/images/repositories/library/dashing/_index_images
2014-06-06 04:25       150   s3://docker-registry/images/repositories/library/dashing/json
2014-06-06 04:25        64   s3://docker-registry/images/repositories/library/dashing/tag_latest
2014-06-06 04:25       150   s3://docker-registry/images/repositories/library/dashing/taglatest_json
```

### まとめ

DockerレジストリのバックエンドはRiakCSを使っているので、容量を気にせずイメージ`docker push`できます。
docker-regitryは5000ポートを開けていますが、VLAN以外のアクセスを拒否しているので、安全にプライベートなイメージを公開できます。

あとでGCEにもデプロイする場合、ベーシック認証とSSLを実装するためにフロントにHTTPサーバーが必要になりそうです。

次回はVLAN内に作成したCoreOSのインスタンス上で、docker-registryから`docker pull`します。
CoreOSがrebootしてもコンテナが起動しているように、Dockerの本番環境を構築します。
