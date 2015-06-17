title: 'HexoにRSS2.0フィードを追加する'
date: 2014-06-23 00:48:08
tags:
 - Hexo
 - RSS
description: Hexoの新しいテーマを決めかねているので、すこし気分転換にRSSフィードを追加しました。_configにキーは定義はしてあったですが、XMLを自動生成していなかったのでプラグインをインストールします。
---

Hexoの新しいテーマを決めかねているので、すこし気分転換にRSSフィードを追加しました。
`_config`にキーは定義はしてあったですが、XMLを自動生成していなかったのでプラグインをインストールします。


<!-- more -->

### hexo-generator-feedプラグイン

[hexo-generator-feed](https://github.com/hexojs/hexo-generator-feed)をインストールして、RSSフィードのXMLを自動生成するようにします。

``` bash
$ cd ~/workspace/blog
$ npm install hexo-generator-feed --save
```

npmでかんたんです。package.jsonは`--save`オプションでバージョンがついています。

``` json ~/workspace/blog/package.json
{
  "name": "hexo-site",
  "version": "2.7.1",
  "private": true,
  "dependencies": {
    "hexo-renderer-ejs": "*",
    "hexo-renderer-stylus": "*",
    "hexo-renderer-marked": "*",
    "hexo-generator-sitemap": "^0.1.4",
    "hexo-generator-feed": "^0.1.2"
  }
}
```

### 設定

プロジェクトの`_config.yml`にRSSフィードのタイプを指定します。今回はRSS2.0を生成します。

``` yml ~/workspace/blog/_config.yml
feed:
    type: rss2
    path: rss2.xml
    limit: 20
```

テーマの`_config.yml`には、`rss`キーに`<header><link>`に追加されるパスを指定します。

``` html
<link rel="alternative" href="/rss2.xml" title="masato's blog" type="application/atom+xml">
```

また、menuにRSSのXMLへのリンクを追加します。

``` yml ~/workspace/blog/themes/biture/_config.yml
# Header
menu:
  RSS: /rss2.xml

rss: /rss2.xml
```
