title: 'Hexoのcurly bracesエスケープ'
date: 2014-06-10 15:36:09
tags:
 - Hexo
 - Markdown
 - Jinja
 - Go
 - AngularJS
description: Hexoには便利なPluginsがいくつか用意されています。hexo-generator-sitemapを使いsitemap.xmlを更新して、検索エンジンにインデックスしてもらいます。
---

[Jinja](http://jinja.pocoo.org/)やGoのtext/template、AngularJSでは、変数のプレースホルダに`double curly braces`を使います。
Hexoのコードブロック内で`double curly braces`使うとエラーになってレンダーできません。

HTMLエンティティの値は、 "{" は `&#123;`、 "}" は `&#125;`です。

コードブロックで使うときは、以下のように書くと、
`"echo 'packer' | &#123;&#123; .Vars &#125;&#125; sudo -E -S sh '&#123;&#123; .Path &#125;&#125;'"`

エラーにならずに表示できます。

```
"echo 'packer' | &#123;&#123; .Vars &#125;&#125; sudo -E -S sh '&#123;&#123; .Path &#125;&#125;'",
```

エラーになると、`hexo server`が起動しなくなるので忘れないようにします。
