title: 'Nitrous.IOのタイムゾーン'
date: 2014-05-01 00:10:26
tags:
 - Hexo
 - NitrousIO
 - GUI開発環境
description: Hexoでブログを書くときは、hexo newコマンドを使いますが、Nitrous.IOで書いているとタイムゾーンがUTCのため、投稿時間がJSTとずれてしまいます。Qiitaに記事があったので、参考にしました。
---
Hexoでブログを書くときは、`hexo new`コマンドを使いますが、Nitrous.IOで書いているとタイムゾーンがUTCのため、投稿時間がJSTとずれてしまいます。
Qiitaに[記事](http://qiita.com/satton_maroyaka/items/961b1873797b0a13f1e6)があったので、参考にしました。
<!-- more -->
環境変数TZを設定します。
``` bash
$ echo "export TZ=Asia/Tokyo" >> ~/.bash_profile
$ source ~/.bash_profile
$ date                                                                        
Sun May 01 00:05:47 JST 2014
```
`hexo new`コマンドを実行したとき、dateがJSTで設定されるようになりました。

``` bash
$ cd ~/workspace/blog
$ hexo new "Nitrous.IO TZ2"
```

``` markdown ~/workspace/blog/source/_posts/nitrousio-tz.md
title: 'Nitrous.IO TZ'
date: 2014-05-01 00:10:26
tags:
```



