title: "Hexoを3.0.1にアップデートする"
date: 2015-04-21 22:48:43
tags:
---

## Hexoのアップデート

npmでHexoを2.8.3から3.0.1にアップデートします。

最初に古いhexoコマンドを削除します。

``` bash
$ rm ~/.parts/bin/hexo
```

新しく`hexo-cli`パッケージをグローバルにインストールします。

``` bash
$ cd
$ npm install hexo-cli -g
```

アップデート用のブログを作成します。

``` bash
$ cd workspace
$ hexo init new-blog
INFO  Copying data to ~/workspace/new-blog
INFO  You are almost done! Don't forget to run 'npm install' before you start blogging with Hexo!
```

package.jsonが自動生成されます。

``` json ~/workspace/new-blog/package.json
{
  "name": "hexo-site",
  "version": "0.0.0",
  "private": true,
  "hexo": {
    "version": ""
  },
  "dependencies": {
    "hexo": "^3.0.0",
    "hexo-generator-archive": "^0.1.0",
    "hexo-generator-category": "^0.1.0",
    "hexo-generator-index": "^0.1.0",
    "hexo-generator-tag": "^0.1.0",
    "hexo-renderer-ejs": "^0.1.0",
    "hexo-renderer-stylus": "^0.2.0",
    "hexo-renderer-marked": "^0.2.4",
    "hexo-server": "^0.1.2"
  }
}

package.jsonからインストールした後サーバーの起動を確認します。

```
$ cd ~/workspace/new-blog
$ npm install
$ hexo server
INFO  Hexo is running at http://0.0.0.0:4000/. Press Ctrl+C to stop.
```

テーマをコピーします。

``` bash
$ cd ~/workspace
$ cp -R ./blog/themes/apollo/ ./new-blog/themes/
```

_config.ymlをマージします。


sourceをコピーします。

``` bash
$ rm -R ./new-blog/sources/_posts
$ mv ./blog/sources ./new-blog/
```

サーバーを起動して動作を確認します。

``` bash
$ cd ~/workspcade/new-blog
$ hexo server
```

問題がなければblogディレクトリとnew-blogディレクトリを入れ替えます。

``` bash
$ rm -fr ~/workspcade/blog
$ mv ~/workspace/new-blog ~/workspace/blog
```

GitHub Pagesへのデプロイ方法が変わっていました。
[Deployment](http://hexo.io/docs/deployment.html)

``` bash
$ npm install hexo-deployer-git --save
```

RSSフィード

``` bash
$ npm install hexo-generator-feed --save
```

sitemap.xml

``` bash
$ npm install hexo-generator-sitemap
```

tag

``` bash
$ npm install hexo-generator-tag --save
```

archive

``` bash
$ npm install hexo-generator-archive --save
```


``` _config.yml
tag_generator:
  per_page: 10

archive_generator:
  per_page: 10
  yearly: true
  monthly: true
  

# Archives
## 2: Enable pagination
## 1: Disable pagination
## 0: Fully Disable
archive: 2
category: 2
tag: 2

```

apolo theme



publicディレクトリに静的ファイルを生成してGitHub Pagesにデプロイします。

``` bash
$ hexo clean && hexo deploy --generate
```

## テーマを変更

$ git clone https://github.com/iissnan/hexo-theme-next themes/next
cp themes/apollo/source/google7cab88483ef366da.html themes/next/source/
cp themes/apollo/source/favicon.ico themes/next/source/


_config.yml

email: ma6ato@gmail.com
language: default
avatar: /images/profile.jpg
timezone: Asia/Tokyo


theme/next/source/css_valiables/base.styl

$font-family-sans-serif = 'Open Sans','Helvetica Neue','Helvetica','Arial','ヒラギノ角ゴ Pro W3','Hiragino Kaku Gothic Pro','メイリオ', Meiryo,'ＭＳ Ｐゴシック','MS PGothic',sans-serif
$font-family-serif = Georgia, "Times New Roman", serif
$font-family-monospace = "Source Code Pro", Consolas, Monaco, Menlo, Consolas, monospace


$font-family-headings     = Lato, $font-family-sans-serif
$font-family-posts        = Lato, $font-family-sans-serif
$font-family-base         = $font-family-posts

