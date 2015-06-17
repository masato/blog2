title: 'IDCFオブジェクトストレージをs3cmdやbotoで使う'
date: 2014-05-20 23:52:57
tags:
 - IDCFオブジェクトストレージ
 - s3cmd
 - boto
 - docker-registry
 - RiakCS
description: IDCフロンティアのオブジェクトストレージサービスはBashoのRiakCSを採用しています。Riak CS Storage APIは、Amazon S3 APIとの互換性が高くできているので、s3cmdやbotoが使えます。Pythonで書かれたブログラムだとdocker-registryなど、内部でbotoを使うことが多いのでさっそく確認してみます。
---
IDCフロンティアの[オブジェクトストレージサービス](http://www.idcf.jp/cloud/storage/)は[Basho](http://basho.com/)の[RiakCS](https://github.com/basho/riak_cs)を採用しています。
[Riak CS Storage API](http://docs.basho.com/riakcs/latest/references/apis/storage/)は、`Amazon S3 API`との互換性が高くできているので、[s3cmd](https://github.com/s3tools/s3cmd)や[boto](https://github.com/boto/boto)が使えます。
Pythonで書かれたブログラムだと[docker-registry](https://github.com/dotcloud/docker-registry)など、内部でbotoを使うことが多いのでさっそく確認してみます。
<!-- more -->

### s3cmdのインストール
s3cmdをインストールします。

``` bash
$ sudo apt-get install s3cmd
```

`s3cmd --configure`で設定ファイルを作成しますが、エンドポイントがAWSと異なるので`Access Key`と`Secret Key`を入力した後は、とりあえず.s3cfgを保存します。

```
$ s3cmd --configure
```
.s3cfgのエンドポイントを修正します。[エンドポイントの確認](http://www.idcf.jp/cloud/storage/faq/sst_001.html)はコントロールパネルから確認できます。
`curly brace`は説明マーカーなので、実際に入力するときは含まれないです。

``` python ~/.s3cfg
access_key = {確認したAccess Key}
host_base = {確認したエンドポイント}
host_bucket = %(bucket)s.{確認したエンドポイント}
secret_key = {確認したSecret Key}
```

設定が終われば、普通のs3cmdとして使えます。

``` bash
$ s3cmd --help
```

### botoのインストール


これまではvirtualenvを作ってましたが、Dockerを使うことで開発環境にはもう不要になりました。
virtualenvだとSSHや設定ファイルの読み込みにはまったりするので。

Dockerコンテナのシェルを起動して、botoをインストールします。

```
$ docker run -i -t ubuntu /bin/bash
# apg-get install python python-pip
# easy_install pip
# pip install boto
```

~/.botoファイルに接続情報を記入します。`curly brace`は実際には含みません。

``` python ~/.boto
[Credentials]
aws_access_key_id = {確認したAccess Key}
aws_secret_access_key = {確認したSecret Key}

[s3]
host = {確認したエンドポイント}
```

pythonのインタラクティブシェルでテストします。
`my-bucket`のところは、グローバルに一意である必要があるので、使われていないバケット名を指定します。

``` python
# python
>>> import boto
>>> boto.set_stream_logger('idcf')
>>> conn = boto.connect_s3(debug=2)
>>> conn.create_bucket('my-bucket')
>>> conn.get_all_buckets()
```

デバッグ情報を含めて、確認できたと思います。
ライブラリによっては、`boto.connect_s3`でなく、`boto.s3.connection.S3Connection`を使っています。

``` python 
>>> import boto.s3.connection
>>> boto.set_stream_logger('idcf')
>>> conn = boto.s3.connection.S3Connection(debug=2)
>>> conn.get_all_buckets()
```
`boto.s3.connect_to_region`というのもありますが、まだ動かせていません。
次は、.botoを使わなず、引数に渡してみます。

``` python ~/idcf_storage.py
#!/usr/bin/env python
import sys
import boto

def main(argv):

  if len(argv) < 2:
    print("Usage: %s <bucket>" % (argv[0],))
    return 1

  accesskey= '確認したAccess Key'
  secretkey = '確認したSecret Key'
  endpoint = '確認したエンドポイント'

  conn = boto.connect_s3(aws_access_key_id=accesskey,
                       aws_secret_access_key=secretkey,
                       host=endpoint)

  bucket = argv[1]
  conn.create_bucket(bucket)
  conn.get_bucket(bucket)
  cs = conn.get_all_buckets()
  for b in cs:
      print b.name

if __name__ == '__main__':
  sys.exit(main(sys.argv))
```

バケット名を指定して実行します。`curly brace`は実際には含みません。

``` bash
# python ~/idcf_storage.py {グローバルで一意になるバケット名}
```

### まとめ
IDCフロンティアのオブジェクトストレージサービスでも、s3cmdやbotoの互換性が高いことが確認できました。
docker-registryでも使えるかどうか、次に試してみようと思います。


