title: 'Hexoはじめました'
date: 2014-04-27 22:06:16
tags:
 - Hexo
 - NitrousIO
 - Nodejs
description: 静的サイト生成ツールにはいくつかありますが、Hexoを使ってみようと思います。どこからでもブログを書けるように、Nitrous.IOでエディットして、GitHub Pagesにデプロイする環境を作ってみます。
---
静的サイト生成ツールにはいくつかありますが、[Hexo](http://hexo.io/) を使ってみようと思います。
どこからでもブログを書けるように、[Nitrous.IO](https://www.nitrous.io/)でエディットして、[GitHub Pages](https://help.github.com/articles/sitemaps-for-github-pages)にデプロイする環境を作ってみます。

<!-- more -->

### Hexoのセットアップ

Hexoのインストール

``` bash
$ npm install hexo -g
```
プロジェクトディレクトリの作成

``` bash
$ cd workspace/
$ hexo init blog && cd blog
```

### 記事の生成

ひな形の作成

``` bash
$ hexo new "Init Hexo"
```

次のようなひな形が作成されるので、titleを日本語にしたり、tagsを付けたりしてから、本文を書いていきます。

``` markdown ~/workspace/blog/source/_posts/init-hexo.md 
title: 'Init Hexo'
date: 2014-04-28 13:38:29
tags:
---
```

### テーマの作成

有志の方々がGitHub上にたくさんの[Themes](https://github.com/tommy351/hexo/wiki/Themes)を公開しているので、適当なテーマを選びます。
[Pure](http://purecss.io/)を使っている、[biture](https://github.com/kywk/hexo-theme-biture)を選んでみました。

``` bash
$ cd workspace/blog/
$ git clone github.com/kywk/hexo-theme-biture.git themes/biture
```

### GitHub へデプロイ

hexoには、Herokuや`GitHub Pages`に簡単にデプロイできるコマンドが用意されています。
{blog_name}のところは環境にあわせてください。

``` yaml ~/workspace/blog/_config.yml
deploy:
  type: github
  repo: git@github.com:{blog_name}/{blog_name}.github.io.git
```

デプロイする前に、ローカルでプレビューできます。
ファイルの更新をwatchしてくれるので、Markdownを編集して保存すると自動的に反映されます。Gruntを使った[Assemble](http://assemble.io/)よりシンプルです。
``` bash
$ hexo server
```

静的ファイルの生成と、デプロイを行います。

``` bash
$ hexo deploy --generate 
```

### まとめ

`GitHub Pages`やAmazonS3を使えば、簡単に無料でブログが公開できます。
[Top Static Site Generators Comparison](http://staticgen.com/)に、ツールの比較がありますので、気に入った言語のツールを試してみるとおもしろいです。

最近はやりの、Goで書かれている[Hugo](http://hugo.spf13.com/)は、やはりはやりのOnePageなWebサイトが作れるので、別の機会に試してみようと思います。

