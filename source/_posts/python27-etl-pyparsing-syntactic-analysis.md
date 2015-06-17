title: "Python2.7でETL - Part1: pyparsingで構文解析"
date: 2014-07-01 22:20:23
tags:
 - Python
 - pyparsing
 - ETL
 - ActivityLog
 - LambdaArchitecture
description: 既存システムのデータベースに保存されている、ユーザーの行動ログを解析することになりました。自分で管理できるアプリケーションなら、好きなようにイベントログをプログラムから出せますが、既存システムは触れないので、データベースからデータを抽出して意味のあるデータにする必要があります。HaskellやANTRLで構文木を勉強したことはあるのですが、当時は日本語をどう扱えばよいかわかりませんでした。今回は情Pythonのpyparsingを使って、日本語を含むUnicodeの構文解析をしてみます。データ分析の8割は加工とクレンジングと言われますが、地味な作業を淡々とやっていきます。pyparsingは、昔からあるPythonの構文解析ツールなのですが、今回初めてです。2日間試行錯誤しましたが、なんとか形になってきました。
---

既存システムのデータベースに保存されている、ユーザーの行動ログを解析することになりました。
自分で管理できるアプリケーションなら、好きなようにイベントログをプログラムから出せますが、
既存システムは触れないので、データベースからデータを抽出して意味のあるデータにする必要があります。

HaskellやANTRLで構文木を勉強したことはあるのですが、当時は日本語をどう扱えばよいかわかりませんでした。
今回は情Pythonのpyparsingを使って、日本語を含むUnicodeの構文解析をしてみます。

データ分析の8割は加工とクレンジングと言われますが、地味な作業を淡々とやっていきます。

pyparsingは、昔からあるPythonの構文解析ツールなのですが、今回初めてです。
2日間試行錯誤しましたが、なんとか形になってきました。

### TL;DR

まだまだ手直しが必要ですが、とりあえずスパイクはできました。

<!-- more -->

### サンプルコード

```python spike5.py
# -*- coding: utf-8 -*-
from pyparsing import (Suppress,Word,Group,Optional,Dict,
                       oneOf,Forward,Literal,ZeroOrMore,
                       nestedExpr,nums,srange)
import dateutil.parser

LBRACK,RBRACK,COL = map(Suppress,'[]:')
MONTH = oneOf("Jan Feb Mar Apr May Jun Jul Aug Sep Oct Nov Dec")
WEEK =  oneOf("Sun Mon Tue Wed Thu Fri Sat")

num =   Word(nums)
dateStr = (WEEK + MONTH + num +
           num + COL + num + COL + num +
           Literal('JST') + num)

def parseDateString(t):
    date = dateutil.parser.parse(
    "{} {} {} {}:{}:{} {} {}".format(*t)
    ).strftime("%Y%m%d%H%M%S")
    return date
dateStr.setParseAction(parseDateString)

unicodePrintables = u''.join(unichr(c) for c in xrange(65536)
                             if not unichr(c).isspace() and
                             unichr(c) not in [u'\u005b', u'\u005d',
                                               u'\u002c'])

ident = Word(unicodePrintables+'-=_@.').setName('ident')
keyData = ident.setName('key')

dictStr = Forward()
nestedBrackets = nestedExpr('[', ']')
valueData = (Optional(dateStr) + Optional(ident).setName('value') +
             Optional(nestedBrackets).setName('nested'))
itemData = Group(LBRACK + keyData + COL + valueData +  RBRACK).setName('itemData')
dictStr <<  Dict(ZeroOrMore(itemData))

sample = u"""
[ uuid : d83a031b-6ecc-4d0d-96af-ea904dfc3408 ][ firstName : 四季 ][ lastName : 真賀田 ][ address : Address [妃真加島, Aichi 470-3504, Japan] ][ timeZone : Japan ][createdAt : Tue Apr 01 10:50:25 JST 2014 ]
"""

data = dictStr.parseString(sample)
print(data)
print("keys: ", data.keys())
print("key_count: ", len(data.keys()))

print(data['address'][1])
```

### 実行結果

解析結果を辞書にするところまでできましたが、まだネストされた括弧内のデータがカンマ区切りで
うまくリストにできません。

```
$ python spike5.py
[[u'uuid', u'd83a031b-6ecc-4d0d-96af-ea904dfc3408'], [u'firstName', u'\u56db\u5b63'], [u'lastName', u'\u771f\u8cc0\u7530'], [u'address', u'Address', [u'\u5983\u771f\u52a0\u5cf6,', u'Aichi', u'470-3504,', u'Japan']], [u'timeZone', u'Japan'], [u'createdAt', '20140401105025']]
('keys: ', [u'uuid', u'firstName', u'lastName', u'address', u'timeZone', u'createdAt'])
('key_count: ', 6)
妃真加島,
```

どういう設計なのかよくわからないのですが、サンプルデータのような括弧で囲まれたデータが30要素以上、
ネストされていたり、日本語があったり、空白が任意にあったり、かなり自由な感じで苦労しています。


### pyparsingインストール

Dockerコンテナを起動してPythonの開発環境を用意します。

``` bash
$ docker run --name pydev -p 8888 -d -t masato/baseimage:1.9 /sbin/my_init
$ ssh root@172.17.0.2 -i ~/.ssh/my_key
$ su - docker
```

Pythonのバージョンは2.7.6です。

``` bash
$ python -V
Python 2.7.6
```

Ubuntuの開発環境なので、システムワイドにpipでインストールします。
日付処理のユーティリティの`python-dateutil`もインストールします。

``` bash
$ sudo pip install pyparsing python-dateutil
...
  Downloading pyparsing-2.0.2.tar.gz (1.1MB): 1.1MB downloaded
...
  Downloading python-dateutil-2.2.tar.gz (259kB): 259kB downloaded
...
Successfully installed pyparsing python-dateutil
Cleaning up...
```

### Unicodeの扱い

pyparsingが定義しているprintableはASCII文字しか対応していないので、パターンマッチで日本語が扱えません。
[Python - pyparsing unicode characters](http://stackoverflow.com/questions/2339386/python-pyparsing-unicode-characters)を参考にしてUnicode文字列を作成します。

空白文字と、`[],`は除外しました。

``` python
unicodePrintables = u''.join(unichr(c) for c in xrange(65536)
                             if not unichr(c).isspace() and
                             unichr(c) not in [u'\u005b', u'\u005d',
                                               u'\u002c'])
```

### 空白を含む日付フォーマット

日付の書式も空白が含まれるので別途パースを書いて、`%Y%m%d%H%M%S`にフォーマットしています。
他の文字のマッチングの邪魔にならないように、数字にフォーマットするため、一番前に処理をもってきました。

### まとめ

データベースに保存されているユーザーの行動ログを、これからデータ解析できるようにクレンジングしています。
過去データは一度CSVにしてから、とりあえずInfluxDBがある解析サーバーへ転送して確認しようと思います。

Kafkaをゲートウェイにして、InfluxDBとTreasureDataへ分岐して保存させれば、
`Lambda Architecture`みたいにならないかな？と思っています。

