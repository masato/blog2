title: 'Hexoとテーマのアップデート'
date: 2014-06-22 18:26:30
tags:
 - Hexo
 - Nitrous.IO
 - git
description: HexoとThemeのアップデートをします。Themesも新しくしようと思うのですが、影響範囲がよくわからないので、今回はbitrureをアップデートするだけにします。
---

HexoとThemeのアップデートをします。[Themes](https://github.com/hexojs/hexo/wiki/Themes)も新しくしようと思うのですが、
影響範囲がよくわからないので、今回は[bitrure](https://github.com/kywk/hexo-theme-biture)をアップデートするだけにします。

<!-- more -->

### hexoの更新

hexoをアップデートします。2.7.1にあがりました。

``` bash
$ npm update hexo -g
hexo@2.7.1 /home/action/.parts/lib/node_modules/hexo
$ cd workspace/blog
$ npm update
```

package.jsonを編集します。バージョンが上がっていくつか依存関係が追加されました。

``` json ~/workspace/blog/package.json
{
  "name": "hexo-site",
  "version": "2.7.1",
  "private": true,
  "dependencies": {
    "hexo-renderer-ejs": "*",
    "hexo-renderer-stylus": "*",
    "hexo-renderer-marked": "*",
    "hexo-generator-sitemap": "^0.1.4"
  }
}
```

### bitureテーマの更新

テーマに使っている[bitrure](https://github.com/kywk/hexo-theme-biture)も更新します。
`git clone`後に修正している箇所があるので、`git pull`で失敗します。

``` bash
$ cd ~/workspace/blog/themes/biture/
$ git pull
remote: Counting objects: 22, done.
remote: Compressing objects: 100% (20/20), done.
remote: Total 22 (delta 8), reused 11 (delta 2)
Unpacking objects: 100% (22/22), done.
From git://github.com/kywk/hexo-theme-biture
   c5ff02c..a570845  master     -> origin/master
 + b95d803...e903ea6 hexi-kywk  -> origin/hexi-kywk  (forced update)
Updating c5ff02c..a570845
error: Your local changes to the following files would be overwritten by merge:
        layout/_partial/article.ejs
        source/css/biture.css
Please, commit your changes or stash them before you can merge.
Aborting
```

`git stash`して`git pull`後の差分を確認します。

``` bash
$ git stash save
$ git pull
Updating c5ff02c..a570845
Fast-forward
 layout/_partial/article.ejs |  1 +
 source/css/biture.css       | 39 ++++++++++++++++++++-------------------
 2 files changed, 21 insertions(+), 19 deletions(-)
$ git stash show
 _config.yml                          | 17 ++++-------------
 layout/_partial/article.ejs          |  3 ++-
 layout/_partial/google_analytics.ejs |  6 +++++-
 layout/_partial/head.ejs             |  4 ++++
 source/css/biture.css                | 29 +++++++++++++++++++++++++++--
 5 files changed, 42 insertions(+), 17 deletions(-)
```

`git stash pop`して、マージ後コンフリクトを解消します。

``` bash
$ git stash pop
Auto-merging source/css/biture.css
CONFLICT (content): Merge conflict in source/css/biture.css
Auto-merging layout/_partial/article.ejs
Recorded preimage for 'source/css/biture.css'
```

### Nitrous.IO上でプレビュー

Nitrous.IO上でプレビューします。

```
$ hexo clean && hexo server`
```

### まとめ
今回の更新で`_ports/`の下のMardownの解釈がちょっと変わったのか、ヘッダーとコードブロックの前後に、行間を空ける必要がありました。

とりあえず無事更新できたので、新しいテーマを探して試してみようと思います。








