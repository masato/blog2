title: 'IDCFオブジェクトストレージをNode.jsのknoxから使う'
date: 2014-05-25 23:52:57
tags:
 - IDCFオブジェクトストレージ
 - Nodejs
 - knox
 - RiakCS
 - Hexo
description: IDCFオブジェクトストレージへPythonのライブラリから試してみましたが、今日はNode.jsのknoxを使ってみます。Node.jsは久しぶりなので、モダンなコードの書き方をよくわかりません。npmでインストールしたモジュールが`NODE_PATH`が設定されていなくて、昨日はしばらくはまりました。PYTHONPATHとか、GOPATHの使い方もよく忘れるので、モジュール管理のベストプラクティスを身につけたいです。
---
IDCFオブジェクトストレージへ[Pythonのライブラリ](/2014/05/20/idcf-storage/)から試してみましたが、今日はNode.jsの[knox](https://github.com/LearnBoost/knox)を使ってみます。
Node.jsは久しぶりなので、モダンなコードの書き方をよくわかりません。
npmでインストールしたモジュールが`NODE_PATH`が設定されていなくて、昨日はしばらくはまりました。
PYTHONPATHとか、GOPATHの使い方もよく忘れるので、モジュール管理のベストプラクティスを身につけたいです。


<!-- more -->

### knoxのインストール

[前回](/2014/05/24/docker-devenv-nodejs/)用意したDocker開発環境を使います。
`--rm`オプションをつけてコンテナは使い捨てます。
``` bash
$ docker run -t -i --rm masato/baseimage /sbin/my_init /bin/bash
root@9fa3a780dcf0:/#
```

NODE_PATHの確認をします。
``` bash
# echo $NODE_PATH
/usr/local/lib/node_modules
# npm root -g
/usr/local/lib/node_modules
```

knoxをインストールします。
``` bash
# npm install -g knox
```

### IDCFオブジェクトストレージの操作

テストに使うoctocatのpngをダウンロードします。
``` bash
# wget http://www.piwai.info/static/blog_img/murakamicat.png
```

s3cmdから、ファイルを保存するバケットを作成します。
-cオプションを指定して、IDCFオブジェクトストレージの設定ファイルを読み込みます。
``` bash
# s3cmd -c ~/.s3cfg.idcf mb s3://knox-bucket
```

テストプログラムを用意します。
コールバックで`res.resume();`をしないと、PUTでハングしてしまうので忘れずに書きます。
GitHubのREADMEには書いてありますが、ちょっとわかりにくいです。

``` node ~/spike.js
var knox = require("knox");
var fs = require('fs');

var client = knox.createClient({
  key: "{確認したAccess Key}",
  secret: "{確認したSecret Key}",
  bucket: "{s3cmdで作成したバケット}",
  endpoint: "{確認したSecret エンドポイント}"
});

// PUT
client.putFile('murakamicat.png', '/test/murakamicat.png',
             {'Content-Type': 'image/png'},
             function(err, res) {
               res.resume();
               if (200 == res.statusCode) {
                 console.log('アップロード成功');
               }
               else {
                 console.log('ファイルアップロード失敗');
               }
             });
// GET
var file = fs.createWriteStream('murakamicat_get.png');
client.getFile('/test/murakamicat.png',
               function(err, res) {
                 res.on('data', function(data) {
                   file.write(data);
                 });
                 res.on('end', function(chunk) {
                   file.end();
                    console.log('ファイルダウンロード成功');
                 });
```

テストスクリプトを実行します。
``` bash
# nodejs spike.js
ファイルダウンロード成功
アップロード成功
```

s3cmdからファイルのアップロードを確認します。
``` bash
# s3cmd -c ~/.s3cfg.idcf ls s3://knox-bucket/test/
2014-05-25 14:33     84887   s3://knox-bucket/test/murakamicat.png
```

### まとめ
knoxからIDCFオブジェクトストレージの操作を確認できました。
本来の目的はknoxを直接操作するのではなく、knoxを内部で利用している[s3-sync](https://github.com/hughsk/s3-sync)と[hexo-deployer-s3](https://github.com/joshstrange/hexo-deployer-s3)
を使うことでした。次回はようやく静的サイトのホスティングを試してみることにします。



