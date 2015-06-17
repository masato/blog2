title: "HexoのアップデートとテーマをTumblrのApolloに変更する"
date: 2014-09-30 01:48:45
tags:
 - Hexo
 - NitrousIO
 - font
description: 久しぶりにHexoのThemesページをみたら種類が増えていました。GhostやTumblrから移植したテーマもミニマルでいい感じです。CSSのfont-familyを見ると最初から日本語フォントが定義されているテーマもいくつかあり、日本でも使っている人が増えたようです。気分転換にテーマをTumblrのApolloに変更することにしました。
---

久しぶりに[HexoのThemes](https://github.com/hexojs/hexo/wiki/Themes)ページをみたら種類が増えていました。GhostやTumblrから移植したテーマもミニマルでいい感じです。CSSのfont-familyを見ると最初から日本語フォントが定義されているテーマもいくつかあり、日本でも使っている人が増えたようです。

気分転換にテーマをTumblrの[Apollo](https://github.com/joyceim/hexo-theme-apollo)に変更することにしました。


<!-- more -->

### Hexoのアップデート

npmで[Hexo](https://github.com/hexojs/hexo)をアップデートします。2.7.1から2.8.3にあがりました。

``` bash
$ npm update hexo -g
$ cd workspace/blog
$ npm update
```

package.jsonも更新されています。

``` json ~/workspace/blog/package.json
{
  "name": "hexo-site",
  "version": "2.8.3",
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

### 2.8のYAMLパーサー変更点

[YAML parser changed](https://github.com/hexojs/hexo/wiki/Migrating-from-2.7-to-2.8)にあるように、値に半角スペースが入る場合はクォートが必要になりました。bitureテーマでは`languages/default.yml`を修正します。

``` yml ~/workspace/blog/themes/biture/languages/default.yml
#archive_b: Archives: %s
archive_b: "Archives: %s"
```
ポストにも`description`の値に`:`があるとOGPタグの生成に失敗するようになったので修正が必要です。

### テーマをApolloに変更

`git clone`でテーマをインストールします。

``` bash 
$ cd ~/workspace/blog/
$ git clone https://github.com/joyceim/hexo-theme-apollo.git themes/apollo
```

themeをapolloに変更します。

``` yaml ~/workspace/blog/_config.yml
#theme: biture
theme: apollo
```

Nitrous.IOのローカルサーバーを起動して確認します。

``` bash
$ hexo server
[info] Hexo is running at http://0.0.0.0:4000/. Press Ctrl+C to stop. 
```


### テーマの設定

テーマの`_config.yml`を編集します。

``` yml ~/workspace/blog/themes/apollo/_config.yml
# Header
menu:
  Home: /
  Archives: /archives
rss: /rss2.xml
# Content
excerpt_link: Read More
fancybox: true
# Miscellaneous
google_analytics: UA-xxx
favicon: /favicon.ico
```

### フォントを源ノ角ゴシックに変更

`sans-serif`を[源ノ角ゴシック](/2014/08/02/gen-no-kaku-gothic/)に変更します。

CSSのプリプロセッサは[Styl](https://github.com/visionmedia/styl)を初めて使います。デフォルトでも日本語フォントが設定されていました。最近気に入っている源ノ角ゴシック(Noto Sans)に変更します。

``` styl ~/workspace/blog/themes/apollo/source/css/_bass/variables.styl
@import url("//fonts.googleapis.com/css?family=Source+Code+Pro");
//@import url("//fonts.googleapis.com/css?family=Open+Sans:400,700");

// Fonts
//font-sans = 'Open Sans','Helvetica Neue','Helvetica','Arial','ヒラギノ角ゴ Pro W3','Hiragino Kaku Gothic Pro','メイリオ', Meiryo,'ＭＳ Ｐゴシック','MS PGothic',sans-serif
font-sans = "Noto Sans", Verdana, Roboto, "Droid Sans", "游ゴシック", YuGothic, "ヒラギノ角ゴ ProN W3", "Hiragino ```

`@font-face`を追加します。

``` styl ~/workspace/blog/themes/apollo/source/css/style.styl
@font-face
  font-family: "Noto Sans"
  src: url(fonts/NotoSansJP-DemiLight.otf) format("opentype")
```

bitureからフォントをコピーします。

```
$ cp biture/source/fonts/NotoSansJP-DemiLight.otf  apollo/source/css/fonts/
```

### デプロイ

プレビューで確認したあと`github.io`へデプロイします。

``` bash
$ hexo clean && hexo deploy --generate
```