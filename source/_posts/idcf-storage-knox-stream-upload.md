title: 'knox-mpuでNode.jsのStreamからRiakCSへマルチパートアップロードする'
date: 2014-06-21 10:46:23
tags:
 - knox
 - knox-mpu
 - Nodejs
 - idcf-compute-api
 - RiakCS
 - IDCFクラウド
 - IDCFオブジェクトストレージ
description: Node.jsの特徴であるStreamを使い、IDCFクラウドからOVAをエクスポートして、ローカルにファイルを保存しないでそのままIDCFオブジェクトストレージにPUTしてみます。Knoxはマルチパートアップロードをサポートしていないので、リモートのファイルをダウンロードしながらStreamでputしようすると大きなファイルの場合失敗してしまうことが多いです。マルチパートアップロードは複雑な仕様なのですが、s3cmdやfogではサポートされているので使うことができます。
---

Node.jsの特徴であるStreamを使い、IDCFクラウドからOVAをエクスポートして、ローカルにファイルを保存しないでそのままIDCFオブジェクトストレージにPUTしてみます。

Knoxは[マルチパートアップロード](http://aws.amazon.com/jp/blogs/aws/amazon-s3-multipart-upload/)をサポートしていないので、リモートのファイルをダウンロードしながらStreamでputしようすると大きなファイルの場合失敗してしまうことが多いです。

マルチパートアップロードは[複雑な仕様](http://stackoverflow.com/questions/8653146/can-i-stream-a-file-upload-to-s3-without-a-content-length-header)なのですが、s3cmdやfogではサポートされているので使うことができます。


<!-- more -->

### ヘルパースクリプト

[前にも](/2014/06/03/idcf-coreos-install-disk/)使いましたが、idcf-compute-apiを便利につかうためのシェルスクリプトを書きます。

``` bash ~/bin/waitjob
#!/usr/bin/env bash

while :
do
  json=$(idcf-compute-api queryAsyncJobResult --jobid=$1)
  status=$(echo ${json} | jq '.queryasyncjobresultresponse.jobstatus')

  if [ ${status} -eq 0 ]; then
    echo -ne "."
    sleep 10s
  else
    echo -e "\n"
    echo ${json} | jq ".queryasyncjobresultresponse | {jobid: $1,jobresult}"
    break;
  fi
done
```

### IDCFクラウドからテンプレートをダウンロードする

IDCFクラウドでは作成したテンプレートを[ダウンロードするURL](http://www.idcf.jp/cloud/docs/api/user/extractTemplate)を取得するAPIがあります。
URLをNode.jsの引数に渡して、StreamでURLのリクエストとRiakCSへのPUTをパイプしてみます。

``` bash ~/idcf_apps/extract_template.sh
#!/usr/bin/env bash

extract_template() {
  waitjob $(idcf-compute-api extractTemplate --id=xxx \
              --mode=HTTP_DOWNLOAD --zoneid=1 \
             | jq '.extracttemplateresponse.jobid')
}

extract_url() {
  echo $1 | jq 'if (.jobresult | has("temolate")) then .jobresult.template.url else .jobresult end' \
            | sed -e 's/%2F/\//g' \
            | sed -e 's/"//g'
}

json=`extract_template`
url=`extract_url "${json}"`

if [[ "${url}" == http* ]]; then
  node ./upload_stream.js ${url}
else
  echo ${url}
fi
```

### knox-mpuを使う

公式の[aws-sdk-js](https://github.com/aws/aws-sdk-js)の場合、RiakCSのようなS3互はエンドポイントの指定問題でなかなかうまく行かないのですが、knoxでは引数で指定できます。knox-mpuもとてもわかりやすくマルチパートアップロードできます。

GoDaddy証明書の検証に失敗するので無視するオプションを入れます。
wgetでも`--no-check-certificate`を使う場合が最近多く、あとで調べようと思います。

``` node ~/idcf_apps/upload_stream.js
#!/usr/bin/env node

process.env['NODE_TLS_REJECT_UNAUTHORIZED'] = '0';

var url = process.argv[2];
console.log('extract ova from: ' + url);

var knox = require('knox')
  , MultiPartUpload = require('knox-mpu')
  , https = require('https')
  , request = require('request');

var fileName = '{FILE_NAME}'
  , bucketName = '{BUCKET_NAME}'
  , endpointName = 'ds.jp-east.idcfcloud.com';

var client = knox.createClient({
  key: '{ACCESS_KEY}'
, secret: '{SECRET_KEY}'
, bucket: bucketName
, endpoint: endpointName
});

var stream = request(url);
upload = new MultiPartUpload(
  {
    client: client
  , objectName: fileName
  , stream: stream
  },function(err,body){
    console.log(body);
  }
);
```

knox-mpuでNode.jsのStream実行をします。Locationのログはs3ですがRiakCSへPUTできています。
``` bash
$ time ./extract_template.sh
extract ova from: https://xxx.realhostip.com/userdata/xxx.ova
{ Location: 'http://xxx.s3.amazonaws.com/packer-v07.ova',
  Bucket: '{BUCKET_NAME}',
  Key: 'packer-v07.ova',
  ETag: '9ZvyHRw2RbmXwVwCMkIoxg==',
  size: 597514240 }
real    1m50.290s
user    0m14.090s
sys     0m3.979s
```

### wgetしてs3cmdを使う場合と処理時間の比較

今度はOVAファイルをwgetした後に、s3cmdでPUTした場合の時間を計ってみます。

最初にはwgetでOVAをローカルにダウンロードします。
``` bash
$ time wget  --no-check-certificate https://xxx.realhostip.com/userdata/xxx.ova
real    2m15.595s
user    0m2.199s
sys     0m2.307s
```

ローカルにダウンロードしたOVAをs3cmdでRiakCSへPUTします。
``` bash
$ time s3cmd -c ~/.s3cfg.idcf put wget-packer-v07.ova  s3://{BUCKET_NAME}.ds.jp-east.idcfcloud.com/
real    1m55.281s
user    0m8.310s
sys     0m3.346s
```

### PUTしたファイルの確認

最後に2つの方法でPUTしたファイルを確認します。
``` bash
$ s3cmd -c ~/.s3cfg.idcf ls s3://{BUCKET_NAME}
2014-06-21 12:06 597514240   s3://{BUCKET_NAME}/packer-v07.ova
2014-06-21 20:09 597514240   s3://{BUCKET_NAME}/wget-packer-v07.ova
```

### まとめ

クラウドサービスなので時間帯や他のユーザーの処理があるので、PUTの時間はまちまちですが、
Node.jsのStreamを使うと、ファイルをダウンロードしてローカルに保存する手間も省けるので便利に使えます。

* knox-mpu: 1m50s
* wget+s3dmd: 4m10s
