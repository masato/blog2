title: 'Measure All The Things!!とHexoのタグプラグイン'
date: 2014-06-27 23:18:58
tags:
 - MeasureAllTheThings
 - Etsy
 - Hexo
 - NewRelic
description: よくxxx All The Things!というイラストを見かけます。VelocityでEtsyのセッションに参加してから、Measure Anything, Measure Everythingを試しています。InfluxDBやTreasure DataのようなTime Series Databaseに、fluentdやcollectdからバシバシ飛ばして、GrafanaやDashingで表示したいのですが、肝心のデータ収集が進みません。DevOpsの文化的な壁もありますが、All The Things!に拘りすぎかも。
---

よく`xxx All The Things!`というイラストを見かけます。

{% img center /2014/06/27/measure-all-the-things-and-hexo-images/measure-all-the-things.png %}


VelocityでEtsyのセッションに参加してから、[Measure Anything, Measure Everything](http://codeascraft.com/2011/02/15/measure-anything-measure-everything/)を試しています。


InfluxDBや`Treasure Data`のような`Time Series Database`に、fluentdやcollectdからバシバシ飛ばして、GrafanaやDashingで表示したいのですが、肝心のデータ収集が進みません。DevOpsの文化的な壁もありますが、`All The Things!`に拘りすぎかも。

<!-- more -->


### Hexoで画像表示

いままでHexoでブログを書いていて、画像を表示をしたことがなかったので、今回はじめて使ってみました。

[使い方](http://hexo.io/docs/writing.html)を読みながら作業します。

まず、`_config.yml`の設定で、`Asset Folder`を有効にします。
これで`~/workspace/blog/souce`にイメージファイルを配置することができます。

``` yml ~/workspace/blog/_config.yml
post_asset_folder: true
```

この後、`hexo new`コマンドで新しいポストを作成すると、

``` bash
$ hexo new "measure all the things and hexo images"
```

`_posts`にこのポスト用のアセットディレクトリが作成されるので、ここにイメージを配置していきます。

```
~/workspace/blog/source/_posts/measure-all-the-things-and-hexo-images
```

上記のディレクトリに、Nitrous.IOのファイルアップロード機能を使い画像を配置します。

[イメージタグ](http://hexo.io/docs/tag-plugins.html)の書式です。

```
{\% img center /2014/06/27/measure-all-the-things-and-hexo-images/measure-all-the-things.png %}
```

### Hyperbole and a Half

オリジナルは[Hyperbole and a Half](http://hyperboleandahalf.blogspot.jp/)の`Allie Brosh`さんです。

{% youtube CtMh71KFmeo %}

[Hyperbole and a Half](http://www.amazon.co.jp/dp/1451666179)というコミックや、[衣類](http://www.zazzle.com/ickybana5/clean+gifts*)も売っています。

彼女の描くイラストも、彼女自身もとても魅力的です。
