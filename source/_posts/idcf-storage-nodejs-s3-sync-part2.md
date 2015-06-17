title: 'IDCFオブジェクトストレージをNode.jsのs3-syncから使う - Part2'
date: 2014-05-30 0:35:23
tags:
 - IDCFオブジェクトストレージ
 - Nodejs
 - knox
 - s3-sync
 - RiakCS
 - Hexo
description: s3-syncがそのまま使えなかったのでForkしました。S3-Compatibleはなかなか結構大変そうです。とりあえずRiakCSでも動くようにしました。コアライブラリや主要なツールは互換性を気にして作られていますが、使いたいライブラリがコンパチブルかどうかは確認しないといけないようです。
---
s3-syncがそのまま[使えなかった](/2014/05/26/idcf-storage-nodejs-s3-sync/)ので[Fork](https://github.com/masato/s3-sync)しました。S3-Compatibleはなかなか結構大変そうです。
とりあえずRiakCSでも動くようにしました。コアライブラリや主要なツールは互換性を気にして作られていますが、
使いたいライブラリがコンパチブルかどうかは確認しないといけないようです。

<!-- more -->

### s3-syncの修正

index.jsのdiffです。
knoxはコンストラクタでencpointを受け取ってくれますが、s3-syncではdestinationで
`amazonaws.com`をハードコードしています。仕方がないので、endpointが引数にきたら尊重してくれるようにしました。

``` node index.js
@@ -53,10 +53,15 @@ function s3syncer(db, options) {
          ? details.path.slice(1)
          : details.path)
  
 +    var endpoint = options.endpoint
 +                 ? options.endpoint + '/'
 +                 : subdomain + '.amazonaws.com/'
 +
      var destination =
            protocol + '://'
 -        + subdomain
 -        + '.amazonaws.com/'
 +        //+ subdomain
 +        //+ '.amazonaws.com/'
 +        + endpoint
          + options.bucket
          + '/' + relative
  
```

### Forkしたs3-syncを使う

プロジェクトに移動して、s3-syncをアンインストール後に、Forkしたmasto/s3-syncをインストールします。

``` bash
$ cd /root/node_apps/sync
$ npm uninstall s3-sync
$ npm install masato/s3-sync
```

readdirpも使っているのでインストールします。

``` bash
$ npm install readdirp
```

今回使うoctocatをダウンロードします。
``` bash
$ mkdir -p ~/node_apps/sync/tmp
$ cd !$
$ wget -O ./tmp/daftpunktocat-guy.gif https://octodex.github.com/images/daftpunktocat-guy.gif
```

### サンプルプログラムの実行

nodeのプログラムは変更ないです。
``` node ~/node_apps/sync/spike.js
var level = require('level')
  , s3sync = require('s3-sync')
  , readdirp = require('readdirp')

var db = level(__dirname + '/cache')

var files = readdirp({
    root: './tmp'
  , directoryFilter: []
})

var uploader = s3sync(db, {
  key:  "{確認したAccess Key}"
 ,secret: "{確認したSecret Key}"
 ,bucket: "{s3cmdで作成したバケット}"
 ,endpoint: "{確認したSecret エンドポイント}"
 ,concurrency: 16
}).on('data', function(file) {
  console.log(file.fullPath + ' -> ' + file.url)
}).on('end', function() {
  console.log('Done!')
});

files.pipe(uploader)
```

プログラムの実行
``` bash
$ node spike.js
/root/node_apps/sync/tmp/saritocat.png -> https://s3-sync.ds.jp-east.idcfcloud.jp/saritocat.png
/root/node_apps/sync/tmp/daftpunktocat-guy.gif -> https://s3-sync.ds.jp-east.idcfcloud.jp/daftpunktocat-guy.gif
```

s3cmdで確認してみると、IDCFオブジェクトストレージにアップロードされています。
``` bash
$ s3cmd -c ~/.s3cfg.idcf ls s3://s3-sync
2014-05-28 09:07    397726   s3://s3-sync/daftpunktocat-guy.gif
2014-05-28 08:55    326653   s3://s3-sync/saritocat.png
```

### まとめ
今度はs3-syncを使っているhexo-deployer-s3もForkして、
ようやくHexoから`Static Site Generator`がIDCFオブジェクトストレージにデプロイできるか試すことができます。
