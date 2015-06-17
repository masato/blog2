title: 'HexoにGoogle Analyticsを設置する'
date: 2014-05-14 00:35:21
tags:
 - Hexo
 - GoogleAnalytics
description: Hexoで使っているbitureテーマは、Google Analyticsの設置が簡単にできます。
---
Hexoで使っている[biture](https://github.com/kywk/hexo-theme-biture)テーマは、[Google Analytics](http://www.google.com/analytics/)の設置が簡単にできます。

<!-- more -->

### Google Analyticsアカウントの作成
アカウントを作成すると、UAで始まるトラッキングIDが取得できます。
アカウント名は任意で、ウェブサイトのURLに、アクセス解析対象のサイトを入力します。

### トラッキングIDの設定

トラッキングIDを、_config.ymlに設定します。
``` yaml ~/workspace/blog/themes/biture/_config.yml
# Miscellaneous
google_analytics: UA-xxxxxxxx-x
```



### ga.jsの修正

``` javascript ~/workspace/blog/biture/layout/_patial/hexo-google-analytics.md
<% if (theme.google_analytics){ %>
<script type="text/javascript">
  var _gaq = _gaq || [];
  // 拡張リンクのアトリビューション分析用のタグをページに設定する
  var pluginUrl = 
   '//www.google-analytics.com/plugins/ga/inpage_linkid.js';
  _gaq.push(['_require', 'inpage_linkid', pluginUrl]);
  _gaq.push(['_setAccount', '<%= theme.google_analytics %>']);
  _gaq.push(['_trackPageview']);

  (function() {
    var ga = document.createElement('script'); ga.type = 'text/javascript'; ga.async = true;
    // ユーザー属性とインタレスト カテゴリに関するレポートを有効にする
    //ga.src = ('https:' == document.location.protocol ? 'https://ssl' : 'http://www') + '.google-analytics.com/ga.js';
    ga.src = ('https:' == document.location.protocol ? 'https://' : 'http://') + 'stats.g.doubleclick.net/dc.js';
    var s = document.getElementsByTagName('script')[0]; s.parentNode.insertBefore(ga, s);
  })();
</script>
<% } %>
```

### デフォルトのビュー

デフォルトのビューは必須項目になっているので、選択して保存する。
```
アナリティクス設定 > プロパティ設定 > デフォルトのビュー > すべてのウェブサイトのデータ
```

### 拡張リンク アトリビューションを使用する

「拡張リンクのアトリビューション分析用のタグをページに設定する」をオンにする。
```
アナリティクス設定 > プロパティ設定 > 拡張リンク アトリビューションを使用する > オン
```

### ユーザー属性とインタレスト カテゴリに関するレポートを有効にする

ga.jsの修正が終わったら、確認をスキップをクリックします。データが表示されるまで、24 時間待ちます。
```
ユーザー > ユーザーの分布 > サマリー > 確認をスキップ
```
