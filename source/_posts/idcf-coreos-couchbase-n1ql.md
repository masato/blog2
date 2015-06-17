title: "Couchbase on Docker - Part6: N1QLを使ってみる"
date: 2015-01-24 20:42:53
tags:
 - CoreOS
 - Couchbase
 - CouchbaseServer
 - IDCFクラウド
 - Nodejs
 - N1QL
description: 2015年1月12日にリリースされたN1QL DP4を試してみます。N1QLを使うとCouchbaseをSQLライクにクエリすることができます。現在のバージョンはDP4 (Developer Preview 4)です。
---


2015年1月12日に[リリース](http://blog.couchbase.com/n1ql-dp4-is-here)された[N1QL DP4](http://docs.couchbase.com/developer/n1ql-dp3/n1ql-intro.html)を試してみます。N1QLを使うとCouchbaseをSQLライクにクエリすることができます。現在のバージョンはDP4 (Developer Preview 4)です。

<!-- more -->

## CentOSコンテナへインストール

N1QLをCentOSコンテナのインストールします。CentOSの[オフィシャルイメージ](https://registry.hub.docker.com/u/library/centos/)をつかいます。

チュートリアルをブラウザから実行するため、8093ポートをDockerホストにマップしてコンテナを起動します。

``` bash
$ docker pull centos:6.6
$ docker run -it --name n1ql -p 8093:8093  centos:6.6 /bin/bash
```

必要なパッケージとN1QLをインストールします。[Downloads](http://www.couchbase.com/nosql-databases/downloads#PreRelease)ページからRed Hat 6用のパッケージをダウンロードして使います。

``` bash
$ yum install -y wget tar
$ wget -qO- http://packages.couchbase.com/releases/couchbase-query/dp4/couchbase-query_dev_preview4_x86_64_linux.tar.gz | tar -zxv -C ${HOME}
```

## N1QLチュートリアル

### チュートリアルサーバーの起動

Couchbase Serverに接続してチュートリアルサーバーを起動します。

``` bash
$ cd ~/cbq-dp4
$ ./cbq-engine -datastore=http://10.3.0.7:8091/
_time="2015-01-20T07:28:07Z" _level="INFO" _msg="New site created with url http://10.3.0.7:8091/"
_time="2015-01-20T07:28:08Z" _level="INFO" _msg="cbq-engine started" version="cbq-dp4" datastore="http://10.3.0.7:8091/"
```

ブラウザを開いて確認します。

![n1ql_hello.png](/2015/01/24/idcf-coreos-couchbase-n1ql/n1ql_hello.png)


チュートリアルはCouchbaseのページから[オンライン](http://query.pub.couchbase.com/tutorial/#1)で試すこともできます。


### コマンドラインツールの起動

別のシェルからコマンドラインツールを起動します。チュートリアルサーバーと同じサーバー上で実行します。`-engine`フラグにはlocalhostを指定します。

``` bash
$ cd ~/cbq-dp4
$ ./cbq -engine=http://localhost:8093/
cbq>
```

データストアの情報をクエリしてみます。

``` bash
cbq> SELECT * FROM system:datastores 
{
    "requestID": "eee04b16-f680-4d9d-9521-d15f76a76c98",
    "signature": {
        "*": "*"
    },
    "results": [
        {
            "datastores": {
                "id": "http://10.3.0.7:8091/",
                "url": "http://10.3.0.7:8091/"
            }
        }
    ],
    "status": "success",
    "metrics": {
        "elapsedTime": "759.196us",
        "executionTime": "583.695us",
        "resultCount": 1,
        "resultSize": 147
    }
}
```

### beer-sampleバケットをクエリする

`beer-sample`バケットからクエリする前に、PRIMARY INDEXを作成します。`beer-sample`はバックチックで囲う必要があります。

``` bash
cbq> CREATE PRIMARY INDEX ON `beer-sample` ;
{
    "requestID": "917d9bde-706b-4845-b9e5-3637f803e155",
    "signature": null,
    "results": [
    ],
    "status": "success",
    "metrics": {
        "elapsedTime": "824.505438ms",
        "executionTime": "824.346364ms",
        "resultCount": 0,
        "resultSize": 0
    }
}
```

N1QLを使うと、SQLのようにクエリを実行することができます。

``` bash
cbq> SELECT * FROM `beer-sample` WHERE type='beer' LIMIT 2;
{
    "requestID": "0f0857e3-d187-4745-a834-ca9a07709ce7",
    "signature": {
        "*": "*"
    },
    "results": [
        {
            "beer-sample": {
                "abv": 0,
                "brewery_id": "anheuser_busch",
                "category": "North American Lager",
                "description": "",
                "ibu": 0,
                "name": "Michelob Hefeweizen",
                "srm": 0,
                "style": "American-Style Lager",
                "type": "beer",
                "upc": 0,
                "updated": "2010-07-22 20:00:20"
            }
        },
        {
            "beer-sample": {
                "abv": 9,
                "brewery_id": "brasserie_de_l_abbaye_val_dieu",
                "description": "",
                "ibu": 0,
                "name": "Triple",
                "srm": 0,
                "type": "beer",
                "upc": 0,
                "updated": "2010-07-22 20:00:20"
            }
        }
    ],
    "status": "success",
    "metrics": {
        "elapsedTime": "325.203897ms",
        "executionTime": "325.095821ms",
        "resultCount": 2,
        "resultSize": 842
    }
}
```

ブラウザからも同様にクエリを実行することができます。

![n1ql_beer_sample_beers.png](/2015/01/24/idcf-coreos-couchbase-n1ql/n1ql_beer_sample_beers.png)




