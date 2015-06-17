title: 'IDCFオブジェクトストレージをNode.jsのs3-syncから使う'
date: 2014-05-26 19:59:53
tags:
 - IDCFオブジェクトストレージ
 - Nodejs
 - knox
 - s3-sync
 - RiakCS
 - Hexo
description: 昨日使ったknoxをrequireしているs3-syncから、IDCFオブジェクトストレージを試してみます。Node.jsのStream APIを使い、LevelDBへオプションでキャッシュできるライブラリです。
---
[昨日](/2014/05/25/idcf-storage-nodejs/)使ったknoxをrequireしている[s3-sync](https://github.com/hughsk/s3-sync)から、IDCFオブジェクトストレージを試してみます。
Node.jsの[Stream API](http://blog.nodejs.org/2012/12/20/streams2/)を使い、[LevelDB](https://code.google.com/p/leveldb/)へオプションでキャッシュできるライブラリです。

### TL;DR
Node.jsのs3-syncはこのままではS3互換サービスで使えないので、[Part2](/2014/05/30/idcf-storage-nodejs-s3-sync-part2/)で[Fork](https://github.com/masato/s3-sync)して修正することになりました。

<!-- more -->

### 開発環境の準備

disposableなコンテナを起動します。
``` bash
$ docker run -t -i --rm  masato/baseimage /sbin/my_init /bin/bash
root@e81096ada1c4:/#
```

今回からssh-agentを使ってコンテナにSSH接続します。
gcutil を使うと内部で[ssh-agent](https://developers.google.com/compute/docs/instances)を使っているので倣おうと思います。

最初にDockerコンテナのIPアドレスを確認します。

``` bash
$ cd ~/docker_apps/phusion
$ docker inspect e81096ada1c4 | jq -r '.[0] | .NetworkSettings | .IPAddress'
172.17.0.4
```

-Aオプションで秘密鍵を追加してコンテナにSSH接続します。
``` bash
$ eval `ssh-agent`
$ ssh-add ./private_key
$ ssh -A root@172.17.0.4
Warning: Permanently added '172.17.0.4' (ECDSA) to the list of known hosts.
root@e81096ada1c4:~#
```

### サンプルプログラム

コンテナにプロジェクトを作成します。

``` bash
$ mkdir -p ~/node_apps/sync
$ cd !$
```

`/usr/bin/node`にシムリンクがないと s3-syncがインストールできないので、

``` bash
$ npm install s3-sync
This failure might be due to the use of legacy binary "node"
```

`/usr/bin/node`を確認してから、s3-syncをインストールします。

``` bash
$ which node
/usr/bin/node
$ npm install s3-sync level readdirp 
```

s3cmdで同期するバケツを作成します。

``` bash
$ s3cmd -c ~/.s3cfg.idcf mb s3://s3-sync
Bucket 's3://s3-sync/' created
```

サンプルに使う今回のoctcatをダウンロードします。

``` bash
$ mkdir tmp
$ wget -O ./tmp/saritocat.png https://octodex.github.com/images/saritocat.png
```

サンプルプログラムを作成します。

``` node ~/node_apps/sync/spike.js
var level = require('level')
  , s3sync = require('s3-sync')
  , readdirp = require('readdirp');
  
var db = level(__dirname + '/cache');

var files = readdirp({
  root: './tmp'
 ,directoryFilter: []
});

var uploader = s3sync(db, {
  key:  "{確認したAccess Key}"
 ,secret: "{確認したSecret Key}"
 ,bucket: "{s3cmdで作成したバケット}"
 ,endpoint: "{確認したSecret エンドポイント}"
 ,concurrency: 16
}).on('data', function(file) {
  console.log(file.fullPath + ' -> ' + file.url);
}).on('end', function() {
  console.log('Done!');
});

files.pipe(uploader);
```

プログラムを実行しますが、残念ながらs3syncのコンストラクタにあるendpointがknoxまで渡されていないようです。
``` bash
$ node spike.js
/root/node_apps/sync/tmp/saritocat.png -> https://s3.amazonaws.com/s3-sync/saritocat.png
```

### Forkして修正することにする

knoxはS3互換サービスを考慮してできていますが、knoxを使っているライブラリのすべてが同じではありません。
これはbotoでも同様で、S3互換サービスを使う場合に注意が必要です。とりあえずGitHubでForkして修正することにします。

また、2014-05-10の[issue](https://github.com/hughsk/s3-sync/issues/9)に上がっているように、ローカルのファイルを削除しても、S3側で同期されて削除されません。
同期ツールとして重要な機能とコメントがありますが、作者は忙しくて直して欲しいそうです。

### まとめ

IDCFオブジェクトストレージのような、S3互換サービスを使う場合のよくある問題に当たりました。

自分が使いたいなら、オープンソースなのでPRしろと言うことだと思いますが、
botoやknoxの周辺のライブラリも、もうちょっとS3互換サービスを考慮してくれるとうれしいです。

次回はGitHubでForkしたライブラリを`npm install`してから再度チャレンジします。
