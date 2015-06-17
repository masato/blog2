title: "Couchbase on Docker - Part5: Beer sampleデータをNode.jsで表示する"
date: 2015-01-23 20:26:26
tags:
 - CoreOS
 - Couchbase
 - CouchbaseServer
 - IDCFクラウド
 - Nodejs
description: Couchbase Serverクラスタにサンプルデータをロードして、Node.jsのプログラムからデータを表示してみます。サンプルでよく使われるBeer sampleのデータを使います。
---

Couchbase Serverクラスタにサンプルデータをロードして、Node.jsのプログラムからデータを表示してみます。[サンプル](https://github.com/couchbase/couchbase-examples)でよく使われるBeer sampleのデータを使います。

<!-- more -->

## beer-sample.zipのロード

3つ起動しているCouchbase Serverコンテナのどれかに接続します。

``` bash
$ docker exec -it 97a45d86ed2d /bin/bash
```

[couchbase-examples](https://github.com/couchbase/couchbase-examples)のリポジトリにある、beer-sample.zipをダウンロードします。

``` bash
$ wget https://github.com/couchbase/couchbase-examples/raw/master/beer-sample.zip
```

cbdocloaderを使いサンプルデータを`beer-sample`バケットにロードします。

``` bash
$ /opt/couchbase/bin/cbdocloader -u user -p passw0rd -b beer-sample beer-sample.zip
[2015-01-20 13:35:50,684] - [rest_client] [140528307840768] - INFO - existing buckets : [u'default']
[2015-01-20 13:35:50,693] - [rest_client] [140528307840768] - INFO - http://127.0.0.1:8091//pools/default/buckets with param: proxyPort=11211&bucketType=membase&authType=sasl&name=beer-sample&replicaNumber=1&saslPassword=&ramQuotaMB=100
[2015-01-20 13:35:50,784] - [rest_client] [140528307840768] - INFO - existing buckets : [u'default']
.[2015-01-20 13:35:52,908] - [rest_client] [140528307840768] - INFO - existing buckets : [u'beer-sample', u'default']
[2015-01-20 13:35:52,909] - [rest_client] [140528307840768] - INFO - found bucket beer-sample
..bucket creation is successful
.
bucket: beer-sample.zip, msgs transferred...
       :                total |       last |    per sec
 byte  :              2541549 |    2541549 |   687339.7
done
```

## データ確認用のNode.jsコンテナ

[dockerfile/nodejs](https://registry.hub.docker.com/u/dockerfile/nodejs/)Dockerイメージを使いNode.jsのコンテナを起動します。コンテナ内でExpressを起動するためポートをマッピングします。

``` bash
$ docker pull dockerfile/nodejs
$ docker run -it --name couchnode -p 1337:1337 dockerfile/nodejs
```

Node.jsのバージョンは、0.10.35です。

``` bash
$ node -v
v0.10.35
```

## Node.js SDK 2.0 のサンプルアプリ

[Node.js SDK 2.0 チュートリアル](http://docs.couchbase.com/developer/node-2.0/tutorial-intro.html)を読みながらサンプルプリを起動します。

サンプルアプリのリポジトリをcloneして`npm install`します。

``` bash
$ git clone git://github.com/couchbaselabs/beersample-node 
$ cd beersample-node 
$ npm install
```

### デザインドキュメント

Node.jsのコードを使って作成できますが、今回はマニュアルで作成します。developmentのビューにデザインドキュメントを作成して、productionにpublishします。

beerデザインドキュメントを作成してpublishします。

* beer-sample > Views > Create Development View

![design_dev_beer_by_name.png](/2015/01/23/idcf-coreos-couchbase-beersample/design_dev_beer_by_name.png)

MapのView codeを編集します。

```js
function (doc, meta) {
    if (doc.type && doc.type == "beer") {
        emit(doc.name, null);
    }
}
```

Filter Results > Show Resultsをクリックするとサンプルのフィルタ結果を確認できます。

![development_view_by_name.png](/2015/01/23/idcf-coreos-couchbase-beersample/development_view_by_name.png)


同様に、breweryデザインドキュメントを作成してpublishします。

* beer-sample > Views > Create Development View


MapのView codeを編集します。

```js
function (doc, meta) {
    if (doc.type && doc.type == "brewery") {
        emit(doc.name, null);
    }
}
```

### サーバーの起動

ExpressはDockerコンテナで起動しています。サンプルのbeer.jsのIPアドレスをCoreOSホスト仮想マシンのIPアドレスに変更します。

```js:beer.js
...
var config = {
    //connstr : "http://localhost:8091",
    connstr : "http://10.3.0.7:8091",
    bucket : 'beer-sample'
}
```

サーバーを起動を起動します。

``` bash
$ node beer.js
Server running at http://127.0.0.1:1337/
Couchbase Connected
```

ブラウザからCoreOSホスト仮想マシンのIPアドレスを開きます。

http://10.3.0.7:1337/

![couchbase_beer_sample.png](/2015/01/23/idcf-coreos-couchbase-beersample/couchbase_beer_sample.png)


Beersの画面から、beer-sampleバケットにロードした`beer-sample.zip`のデータを表示できました。

![couchbase_beer_sample_beers.png](/2015/01/23/idcf-coreos-couchbase-beersample/couchbase_beer_sample_beers.png)
