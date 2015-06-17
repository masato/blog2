title: "今更ながらTwitterのデータで楽しく遊ぶ - Part1: Commander.jsとlodashとGmail"
date: 2015-06-04 11:20:10
tags:
 - Twitter
 - Twit
 - Nodejs
 - Commanderjs
 - lodash
 - Gmail
description: 今更ながらTwitterのデータを検索して遊んでいます。Twitterのデータはリアルタイム性がありメタデータも豊富に定義されています。世界中の人が勝手につぶやいてくれるので、属性をもつランダムなテキストデータを簡単に取得できます。最初にNode.jsのCommander.jsで簡単なCLIのインタフェースを用意します。REST APIのGET search/tweetsとStreaming APIのPOST statuses/filterをCLIから試しながらどんなデータが取得できるかプロトタイプします。
---

今更ながらTwitterのデータを検索して遊んでいます。Twitterのデータはリアルタイム性がありメタデータも豊富に定義されています。世界中の人が勝手につぶやいてくれるので、属性をもつランダムなテキストデータを簡単に取得できます。最初にNode.jsの[Commander.js](https://github.com/tj/commander.js)で簡単なCLIのインタフェースを用意します。REST APIの[GET search/tweets](https://dev.twitter.com/rest/reference/get/search/tweets)とStreaming APIの[POST statuses/filter](https://dev.twitter.com/streaming/reference/post/statuses/filter)をCLIから試しながらどんなデータが取得できるかプロトタイプします。

<!-- more -->

## プロジェクトと使い方

作成したプロジェクトは[リポジトリ](https://github.com/masato/node-tweet)にpushしています。

```bash
$ cd ~/node_apps/node-tweet
$ tree
.
├── Dockerfile
├── README.md
├── app.js
├── commands
│   └── search.js
├── config.json
├── config.json.original
├── docker-compose.yml
├── node_modules -> /dist/node_modules
├── package.json
├── test.csv
└── utils
    └── gmail.js
```

カレントディレクトリにシムリンクを作ってからbuildします。

```
$ ln -s /dist/node_modules .
$ docker-compose build
```

### 簡単な使い方

[README.md](https://github.com/masato/node-tweet/blob/master/README.md)に例があります。[GET search/tweets](https://dev.twitter.com/rest/reference/get/search/tweets)はREST APIです。以下のように指定すると、`babymetal`のキーワードでドイツ語のtweetを10件検索します。検索結果はCSV形式のファイルにして指定したアドレスに添付為てメールします。

```bash
$ docker-compose run --rm twitter search "babymetal -filter:retweets" -- -l de -C -e ma6ato@gmail.com
```

[POST statuses/filter](https://dev.twitter.com/streaming/reference/post/statuses/filter)はStreaming APIです。以下のように指定すると、`android`のキーワードで英語のtweetをストリームで取得して標準出力します。

```bash
$ docker-compose run --rm twitter track android -- -l en
```

## app.js

app.jsがエントリポイントです。[Commander.js](https://github.com/tj/commander.js)を使いCLIのプログラムを構成します。オプションは[GET search/tweets](https://dev.twitter.com/rest/reference/get/search/tweets)の仕様に加えて、CSVファイル作成と検索結果をGmailで送信する機能のフラグをつけました。

```js ~/node_apps/node-tweet/app.js
'use strict';

var program = require('commander')
  , search = require('./commands/search.js');

program
    .version('0.0.1')

program
    .command('search <query>')
    .option('-c, --count [count]','number of tweets 1-100')
    .option('-r, --result_type [type]','result type')
    .option('-l, --lang [ISO 639-1]','language of tweet')
    .option('-L, --locale [ja]','language of query')
    .option('-e, --email <to>','send email')
    .option('-C, --csv','output csv format')
    .action(search.commandQuery);

program
    .command('track <track>')
    .option('-l, --language [language]','BCP47','language of tweet [ja]','ja')
    .action(search.commandTrack);
    
program.parse(process.argv);

if (process.argv.length < 4) {
    console.log('You must specify a command');
    program.help();
} else if (['search','track'].indexOf(process.argv[2]) < 0){
    console.log(process.argv[2] + 'is not a command');
    program.help();
}

exports.program = program;
```

## search.js

search.jsにコマンドを実装します。特に難しいことはしないで[twit](https://github.com/ttezel/twit)のライブラリを実行しています。ここでの目的はTweetデータの中身から使えそうなフィールドを抽出することと、[lodash](https://lodash.com/)の関数合成や他の関数を試してみることです。

```js ~/node_apps/node-tweet/commands/search.js
'use strict';

var Twit = require('twit'),
    config = require('../config.json'),
    util = require('util'),
    _ = require('lodash'),
    moment = require('moment-timezone'),
    json2csv = require('nice-json2csv'),
    mailer = require('../utils/gmail');

function connect() {
    return new Twit({
        consumer_key: config.twitter_key,
        consumer_secret: config.twitter_secret,
        access_token: config.twitter_token,
        access_token_secret: config.twitter_token_secret
    });
}

function prettyJson(data){
    return JSON.stringify(data,null,2);
}
        
function formatDate(created_date){
    return moment(new Date(created_date)).tz("Asia/Tokyo").format();
}

function createUrl(screen_name,id_str){
    return util.format('https://twitter.com/%s/status/%s',
                       screen_name,id_str);
}

function pluckPath(data,path,key,suffix) {
    return _.pluck(_.get(data,path),key)
            .map(function(x) {return suffix+x})
            .toString();
}

function extractData(s) {
    return {
        id: s.id_str,
        url: createUrl(s.user.screen_name,s.id_str),
        created_at: formatDate(s.created_at),
        lang: s.lang,
        name: s.user.name,
        screen_name: s.user.screen_name,
        user_url: s.user.url || '',
        text: s.text,
        source: s.source,
        expanded_url: _.get(s,'entities.urls[0].expanded_url',''),
        hashtags: pluckPath(s,'entities.hashtags','text',"#"),
        user_mentions: pluckPath(s,'entities.user_mentions','screen_name',"@"),
        retweeted_status: s.retweeted_status ? true : false
    }
}

function searchPrint(csv,email,metadata) {
    return _.compose(email ? _.curry(mailer.sendWithAttachment)(email,metadata) : console.log,
                     csv ? json2csv.convert : prettyJson,
                     _.map);
}

function commandTrack (track,options) {
    connect()
        .stream('statuses/filter', {track: track, language: options.language})
        .on('tweet', _.compose(console.log, prettyJson, extractData));
}

function commandQuery (query,options) {
    connect()
        .get('search/tweets',{ q: util.format('%s lang:%s', query, options.lang),
                               locale: options.locale,
                               count: options.count,
                               result_type: options.result_type},
             function(err, data, response) {
                 if (err) return console.log(err);
                 searchPrint(options.csv, options.email,
                             _.pick(data.search_metadata,
                                    ['since_id_str', 'max_id_str', 'query']))
                 (data.statuses, extractData);
             });
}

module.exports = {
    commandQuery: commandQuery,
    commandTrack: commandTrack
}
```

### Commander.jsのoptionは難しい

Commander.jsの引数の扱い方はちょっと癖があるので思った通りの引数がうまく取得できません。CLIからオプションが未指定の場合は、action()に指定したコマンドの引数のoptionsにundefinedで入ります。リクエストを発行する関数に渡すparamsのオブジェクトは動的に作成することになります。[Conditionally set a JSON object property](http://stackoverflow.com/questions/18019854/conditionally-set-a-json-object-property)によるとstringifyでJSONを文字列にするときにundefinedなフィールドは作成しないようです。

[twitter.js](https://github.com/ttezel/twit/blob/master/lib/twitter.js)でもparamsはstringfyしているのでCommander.jsのoptionsはundefinedなフィールドをそのまま渡しました。

```js
var paramsClone = JSON.parse(JSON.stringify(params))
```

`--result_type`オプションの場合、オプションが1つだけなら`<type>`と記述するとUNIX的に入力が必要なコマンドになります。

```bash
$ docker-compose run --rm twitter search "babymetal -filter:retweets" -- --result_type
  error: option `-r, --result_type <type>' argument missing
```

ただし複数のオプションがあると必須が効かなくなり意図しない動作をします。以下の場合`recent_type`には次のオプションの`-l`が入ってしまします。

```bash
$ docker-compose run --rm twitter search "babymetal -filter:retweets" -- --result_type -l ja
```

また最後の引数に`recent`としてデフォルト値を使う場合、コマンドラインから`--recent_type`が未指定でも`options.result_type`の変数に`recent`が入ってしまいます。未指定の場合は`undefined`を意図しているのでこれも困ります。

```js
    .option('-r, --result_type [type]','result type','recent')
```

試行錯誤すると以下のように`[type]`としてデフォルト値を使わないのがよさそうです。

```js
    .option('-r, --result_type [type]','result type')
```

`--result_type`を指定しない場合、`options.result_type`は`undefined`になるためparamsをstringifyするときには削除されます。

### lodashで関数型言語

Node.jsを関数型言語っぽく書くために[lodash](https://lodash.com/)を使っています。ほぼこのプログラムを書いた主な目的になっています。APIで取得したTwitterのリストを加工するときに関数型言語のように扱ってみます。

関数合成ができるできるcomposeの[flowRight](https://lodash.com/docs#flowRight)が便利です。以下のように`on('tweet')`のコールバックに`_.compose`で関数合成した関数を指定できます。`on('tweet')`はコールバックの引数にTweetのデータを渡します。右から順番に`extractData`でJSONから必要なフィールドを抽出して、`return`を`JSON.stringify`に渡し、さらにその`return`を標準出力に渡します。

```js
function commandTrack (track,options) {
    connect()
        .stream('statuses/filter', {track: track, language: options.language})
        .on('tweet', _.compose(console.log, prettyJson, extractData));
}
```

## gmail.js

このモジュールでは[Nodemailer](https://github.com/andris9/Nodemailer)で添付ファイルをGmailで送信しています。この例ではメールアドレスのバリデーションや、メール本文にTweetのメタデータを追加しています。Nodemailerを使うとGmailの添付ファイル送信がとても簡単に書くことができます。

```js ~/node_apps/node-tweet/utils/gmail.js
'use strict';

var nodemailer = require('nodemailer'),
    validator = require('validator'),
    config = require('../config.json'),
    util = require('util'),
    moment = require('moment-timezone');

function createFilename() {
    return util.format('twitter-%s.csv',
                       moment(new Date())
                       .tz("Asia/Tokyo")
                       .format('YYYYMMDD-HHmmss'));
}

function formatMetadata(data) {
    return ["query:'",decodeURIComponent(data.query.replace(/\+/g,' ')),"'\n",
            "since_id: ",data.since_id_str,"\n",
            "max_id: ",data.max_id_str,"\n"].join('');
}

function sendWithAttachment(gmail_to,metadata,content) {
    if (! validator.isEmail(gmail_to)) {
        return console.log('--email is not valide: ', gmail_to);
    }

    var transporter = nodemailer.createTransport({
        service: 'Gmail',
        auth: {
            user: config.gmail_user,
            pass: config.gmail_pass
        }
    });

    var msg = {
        from: config.gmail_from,
        to: gmail_to,
        subject: config.gmail_subject,
        text: formatMetadata(metadata),
        attachments: [
            {filename: createFilename(),
             content: content}
        ]
    };

    transporter.sendMail(msg, function (err) {
      if (err) {
        console.log('Sending to ' + msg.to + ' failed: ' + err);
      }
      console.log('Sent to ' + msg.to);
    });
}

module.exports.sendWithAttachment = sendWithAttachment;
```

## まとめ

今回のプロトタイプでTwitterのAPIにどのようなパラメータがあり、Tweetデータからどのようなフィールドを取得して加工すると有用なのか使いながらフィードバックを得ることができます。次はジョブをキューしたり、定期的に処理をスケジュール管理できるようにしてみます。