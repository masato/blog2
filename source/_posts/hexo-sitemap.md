title: 'Hexoでsitemap.xmlを自動更新する'
date: 2014-05-08 10:30:08
tags:
 - Hexo
description: Hexoには便利なPluginsがいくつか用意されています。hexo-generator-sitemapを使いsitemap.xmlを更新して、検索エンジンにインデックスしてもらいます。
---
Hexoには便利な[Plugins](https://github.com/tommy351/hexo/wiki/Plugins)がいくつか用意されています。
[hexo-generator-sitemap](https://github.com/hexojs/hexo-generator-sitemap)を使いsitemap.xmlを更新して、検索エンジンにインデックスしてもらいます。

<!-- more -->

### インストール
hexo-generator-sitemapプラグインをインストールします。
``` bash
$ npm install hexo-generator-sitemap --save
```

### sitemap.xmlの作成と設置
generateすると、最新の投稿の状態でpublic/sitemap.xmlが再作成されます。
``` bash
$ hexo generage
```

deployすると、[sitemap.xml](http://masato.github.io/sitemap.xml)は、index.htmlと同じトップディレクトリに設置されます。
``` bash
$ hexo deploy
```
