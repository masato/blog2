title: 'IDCFオブジェクトストレージでdocker-registryを使う'
date: 2014-05-22 12:35:23
tags:
 - IDCFオブジェクトストレージ
 - s3cmd
 - boto
 - Docker
 - docker-registry
 - RiakCS
description: 前回、botoからRiakCSで構築されているIDCFオブジェクトストレージに接続ができました。次はdocker-registryを使い、IDCFオブジェクトストレージ上にDockerイメージのリポジトリを構築してみます。Dokerを使った開発に慣れてきたので、そろろろ開発環境のコンテナを別のクラウドにデプロイしたいのですが、index.docker.ioを使わずにプライベートでできるとテストしやすいです。Packerで作ったイメージの保管先にもできると思います。
---

* `Update 2014-08-10`: [Dockerでデータ分析環境 - Part7: Ubuntu14.04のDockerにIPython Notebookをデプロイ](/2014/08/10/docker-analytic-sandbox-ipython-notebook-deploy/)


[前回](/2014/05/20/idcf-storage/)、botoからRiakCSで構築されているIDCFオブジェクトストレージに接続ができました。次は[docker-registry](https://github.com/dotcloud/docker-registry)を使い、IDCFオブジェクトストレージ上にDockerイメージのリポジトリを構築してみます。Dokerを使った開発に慣れてきたので、そろろろ開発環境のコンテナを別のクラウドにデプロイしたいのですが、[index.docker.io](https://index.docker.io/)を使わずにプライベートでできるとテストしやすいです。[Packer](http://www.packer.io/)で作ったイメージの保管先にもできると思います。

<!-- more -->

[docker-registry](https://github.com/dotcloud/docker-registry)のREADMEにS3互換のオブジェクトストレージのユースケースが書いてありますが、botoのどのAPIを使っているか確認してみます。

### バケットを作成

最初にDockerイメージを保管するバケットを、IDCFオブジェクトストレージに作成します。

``` bash
$ s3cmd mb s3://docker-registry
Bucket 's3://docker-registry/' created
```

### botoの使い方

config.ymlの設定ファイルは、docker-registryコンテナを起動するときにDockerボリュームでマウントして使います。
先に完成した設定ファイルです。

``` yml ~/registry_conf/config.yml
common:
    loglevel: info
    secret_key: _env:REGISTRY_SECRET
    standalone: true
    disable_token_auth: true
 
dev:
    storage: s3
    loglevel: debug
    s3_access_key: _env:AWS_S3_ACCESS_KEY
    s3_secret_key: _env:AWS_S3_SECRET_KEY
    s3_bucket: docker-registry
    boto_bucket: docker-registry
    boto_host: ds.jp-east.idcfcloud.com
    s3_encrypt: false
    s3_secure: true
    storage_path: /images
```

IDCFオブジェクトストレージを使うときは、`boto_host`の設定が必用です。

docker_registry.coreの[boto.py](https://github.com/dotcloud/docker-registry/blob/master/depends/docker-registry-core/docker_registry/core/boto.py)を見ると以下のようになっています。`.boto`に記述してもみてくれないようです。

``` python boto.py
    def _build_connection_params(self):
        kwargs = {'is_secure': (self._config.boto_secure is True)}
        config_args = [
            'host', 'port', 'debug',
            'proxy', 'proxy_port',
            'proxy_user', 'proxy_pass'
        ]
        for arg in config_args:
            confkey = 'boto_' + arg
            if getattr(self._config, confkey, None) is not None:
                kwargs[arg] = getattr(self._config, confkey)
        return kwargs
```



### docker-registryコンテナの起動

docker-registryコンテナは、IDCFオブジェクトストレージのゲートウエイとして機能します。

-eオプションで渡す環境変数は、REGISTRY_SECRETには、base64エンコードされた文字列を設定します。
AWS_S3_ACCESS_KEYとAWS_S3_SECRET_KEYには、それぞれコントロールパネルで確認した値を使います。


REGISTRY_SECRET用にランダムを作成するため、ホームディレクトリの`.rnd`を削除します。

```
$ sudo rm ~/.rnd
```

`docker run`で、`registry`イメージを使い起動します。

``` bash
$ docker run -p 5000:5000 -v /home/masato/registry_conf:/registry_conf -e DOCKER_REGISTRY_CONFIG=/registry_conf/config.yml -e AWS_S3_ACCESS_KEY="確認したAccess Key" -e AWS_S3_SECRET_KEY="確認したSecret Key" -e REGISTRY_SECRET=`openssl rand -base64 64 | tr -d '\n'` registry
```

localhost:5000へポートフォワードしているので、Dockerホストからcurlで確認します。

``` bash
$ curl localhost:5000
"docker-registry server (dev) (v0.6.9)"
```

### PUSHとPULL

ローカルに保存しているイメージにタグをつけます。書式は以下です。

```
docker tag IMAGE REPOSITORY[:TAG]
```

`docker tag`でubuntuイメージいにリポジトリ名とタグを付けます。

``` bash
$ docker tag ubuntu localhost:5000/ubuntu
```

`docker push`でIDCFオブジェクトストレージのリポジトリにpushします。

``` bash
$ docker push localhost:5000/ubuntu
The push refers to a repository [localhost:5000/ubuntu] (len: 1)
Sending image list
Pushing repository localhost:5000/ubuntu (1 tags)
511136ea3c5a: Image successfully pushed
5e66087f3ffe: Image successfully pushed
4d26dd3ebc1c: Image successfully pushed
d4010efcfd86: Image successfully pushed
99ec81b80c55: Image successfully pushed
Pushing tag for rev [99ec81b80c55] on {http://localhost:5000/v1/repositories/ubuntu/tags/latest}
```

s3cmdを使い、IDCFオブジェクトストレージに保管されていることを確認します。

``` bash
$ s3cmd ls s3://docker-registry/images/repositories/library/ubuntu/
2014-05-22 01:35       380   s3://docker-registry/images/repositories/library/ubuntu/_index_images
2014-05-22 01:35       150   s3://docker-registry/images/repositories/library/ubuntu/json
2014-05-22 01:35        64   s3://docker-registry/images/repositories/library/ubuntu/tag_latest
2014-05-22 01:35       150   s3://docker-registry/images/repositories/library/ubuntu/taglatest_json
```

`docker pull`と`docker run`でイメージの起動を確認してみます。

```
$ docker pull localhost:5000/ubuntu
Pulling repository localhost:5000/ubuntu
99ec81b80c55: Download complete
511136ea3c5a: Download complete
5e66087f3ffe: Download complete
4d26dd3ebc1c: Download complete
d4010efcfd86: Download complete
$ docker run -t -i localhost:5000/ubuntu /bin/bash
root@2c3c0359ca72:/#
```

### まとめ

IDCFオブジェクトストレージでも簡単にDockerリポジトリを作ることができました。
Packerでどんどんイメージを作りながら、GCEのCoreOSやHerokuにデプロイするのが目標です。

docker-registryは、Pythonで作られている、Gevent,Gunicorn,Flaskで動くWSGIアプリです。
おもしろそうなのでソースコードも読んでいきたいと思います。[Bugsnag](https://bugsnag.com/)というエラー監視サービスも初めて知りました。あたらしいツールをつかっていると、いろいろ勉強になることが多いです。

