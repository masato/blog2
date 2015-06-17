title: "Node.jsでlodashのflowRightを使って関数合成をする"
date: 2015-05-27 23:11:10
tags:
 - Nodejs
 - lodash
 - 関数合成
 - Clojure
 - Twitter
description: Clojureで関数合成が楽しくなると、Node.jsでも関数型言語っぽく使いたくなります。npmのライブラリにはUnderscore.js、lodash、Ramdaなどあります。ググってみるとlodashが良さそうなので関数のcomposeをしてみます。
---

Clojureで関数合成が楽しくなると、Node.jsでも関数型言語っぽく使いたくなります。npmのライブラリには[Underscore.js](http://underscorejs.org/)、[lodash](https://lodash.com/)、[Ramda](http://ramdajs.com/)などあります。ググってみると[lodash](https://lodash.com/)が良さそうなので関数のcomposeをしてみます。

<!-- more -->

## Clojureのcomp

Clojureでは[comp](https://clojuredocs.org/clojure.core/comp)を使って簡単に関数合成ができます。

```clj
((comp str +) 8 8 8)
;;=> "24"
```

## lodashのflowRight

[lodash](https://lodash.com/)Underscore.js互換のライブラリでより多機能になっています。[flowRight](https://lodash.com/docs#flowRight)を使って関数合成ができます。`_.backflow`と`_.compose`のエイリアスがついているので個人的に好きな`_.compose`を使います。

ちょうど[twit](https://github.com/ttezel/twit)と[Commander.js](https://github.com/tj/commander.js)を使ってTwitter検索ツールを開発しているところなので、取得したTweetのデータを標準出力するところで使ってみます。lodashのバージョンは3.9.3です。以下はCommander.jsのコマンド処理のコードの抜粋です。

```js ~/node_apps/node-tweet/commands/search.js

function prettyJson(data){
    return JSON.stringify(data,null,2);
}

function extractData(s) {
    return {
        id: s.id_str,
        url: createUrl(s.user.screen_name,s.id_str),
        created_at: formatDate(s.created_at),
        name: s.user.name,
        screen_name: s.user.screen_name,
        user_url: s.user.url || '',
        text: s.text,
        source: s.source,
        expanded_url: _.get(s,'entities.urls[0].expanded_url','')
    }
}

function prettyPrint(csv) {
    return _.compose(console.log,
                     csv ? json2csv.convert : prettyJson,
                     _.map);
}

function commandQuery (query,count,options) {
    connect()
        .get('search/tweets',
             { q: util.format('%s lang:ja',query),
               count: count ? parseInt(count) : 10 },
             function(err, data, response) {
                 if (err) return console.log(err);
                 prettyPrint(options.csv)(data.statuses, extractData);
             });
}
```

関数合成に関係ないコードもありますが、`_.compose`はprettyPrint関数で使っています。

```js
function prettyPrint(csv) {
    return _.compose(console.log,
                     csv ? json2csv.convert : prettyJson,
                     _.map);
}
```

flowRightはClojureのcompと同様に右側から順番に関数を評価する新しい関数を生成します。この例では_.mapでextractData関数をあててJSONを整形してから、CSVオプションが引数にある場合は[nice-json2csv](https://github.com/matteofigus/nice-json2csv)を使ってCSV文字列に変換します。そうでなければ2タブでstringifyします。最後にconsole.logを実行して標準出力しています。CSVフラグのように関数合成の途中で三項演算子も式として使えます。
