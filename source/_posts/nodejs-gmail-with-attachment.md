title: "Node.jsからGmailの添付ファイルを送信する"
date: 2015-06-01 23:33:18
tags:
 - Nodejs
 - Gmail
 - lodash
 - Nodemailer
 - 関数型言語
description: 誰かにファイルを送信する場合、今の世の中でも普通の人はフォーマットはCSVで方法はメールで欲しいというのがよくあります。スケジュールで自動的にGmailを使って添付メールを送りたかったのでNodemailerを使って書いてみました。本当はちゃんとした内容の添付ファイルなのですが、テスト用にランダム文字列をChance.jsで生成しました。最近lodashが気に入っているので関数型言語っぽい書き方です。
---

誰かにファイルを送信する場合、今の世の中でも普通の人はフォーマットはCSVで方法はメールで欲しいというのがよくあります。スケジュールで自動的にGmailを使って添付メールを送りたかったので[Nodemailer](http://www.nodemailer.com/)を使って書いてみました。本当はちゃんとした内容の添付ファイルなのですが、テスト用にランダム文字列を[Chance.js](http://chancejs.com/)で生成しました。最近[lodash](https://lodash.com/)が気に入っているので関数型言語っぽい書き方です。

<!-- more -->

## プロジェクト

作成したサンプルプログラムは[リポジトリ](https://github.com/masato/node-gmail)に置いています。

```bash
$ cd /node_apps/node-gmail
$ tree -L 1
.
├── Dockerfile
├── README.md
├── app.js
├── config.json
├── config.json.original
├── docker-compose.yml
├── gmail.js
├── node_modules -> /dist/node_modules
└── package.json
```

開発と実行はDocker Composeを使っています。カレントディレクトリに`/dist/node_modules`へのシムリンクを作っておきます。`/dist/node_modules`はDockerホスト内には存在しませんが、このディレクトリをコンテナにマウントしたときにコンテナ内で有効になります。

```bash
$ ln -s /dist/node_modules .
```

## プログラム

app.jsがエントリポイントになります。

```js /node_apps/node-gmail/app.js
'use strict';

var _ = require('lodash'),
   json2csv = require('nice-json2csv'),
   mailer = require('./gmail'),
   Chance = require('chance'),
   chance = new Chance(),
   args = process.argv.splice(2),
   email = args[0],
   data = _.times(10, function(n){
       return _.zipObject(_.map(['a','b','c'],
                                function(v){return [v,chance.name()]}))});

function prettyPrint(email) {
    return  _.compose(_.curry(mailer.sendWithAttachment)(email),
                      json2csv.convert,
                      _.map)}
prettyPrint(email)
           (data, function(s){return _.pick(s,['a','c'])});
```

[lodash](https://lodash.com/)を使ってなるべく変数を使わずに関数型言語っぽく書いてみました。プログラムの実行にはあまり影響はないです。Node.jsだとコールバックにfunctionとreturnの宣言が必要なのでちょっと長くなります。

ランダムな値を[Chance.js](http://chancejs.com/)で作成して`a`,`b`,`c`をキーに持つマップを10個作成します。

```js
data = _.times(10, function(n){
       return _.zipObject(_.map(['a','b','c'],
                                function(v){return [v,chance.name()]}))});
```

prettyPrint関数で生成する関数合成の最初の`_map`では上記の`data`に`_.pick(s,['a','c'])`を適用して`a`と`c`のキーと値だけ抽出しています。

```js
prettyPrint(email)
           (data, function(s){return _.pick(s,['a','c'])});
```

関数合成では右から順に`_.map`、`json2csv.convert`、カリー化した`mailer.sendWithAttachment`と評価されていきます。


## 添付ファイル付きのGmail

`app.js`で実行している`sendWithAttachment`関数は`gmail.js`から`module.exports`しています。

```js /node_apps/node-gmail/gmail.js
'use strict';

var nodemailer = require('nodemailer'),
    validator = require('validator'),
    config = require('./config.json'),
    util = require('util'),
    moment = require('moment-timezone');

function createFilename() {
    return moment(new Date())
             .tz("Asia/Tokyo")
             .format('YYYYMMDD-HHmmss') + '.csv';
}

function sendWithAttachment(gmail_to,content) {
    console.log(content);
    if (! validator.isEmail(gmail_to)) {
        return console.log('--email is not valid: ', gmail_to);
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
        text: config.gmail_text,
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

添付ファイルの中身は`content`変数に文字列としてCSVデータが入っています。`config.json`にGmailの認証情報とメールの件名などの設定を記述して実行します。`docker-compose run`を実行すると引数に指定したメールアドレスに添付ファイル付きのメールが送信されます。添付ファイルの内容は確認用に標準出力もしています。

```bash
$ docker-compose run --rm gmail -- ma6ato@gmail.com

> node-gmail@0.0.1 start /app
> node app.js "ma6ato@gmail.com"

"a","c"
"Dean Luna","Minerva Walker"
"Dylan Richards","Alvin Bryant"
"Oscar Martin","Norman Dean"
"Elmer Long","Leroy Phelps"
"Caroline Hayes","Lura Foster"
"Belle Clarke","Dora Bell"
"Garrett McDonald","Craig Grant"
"Hester Silva","Donald Harris"
"Gertrude Boone","Jennie Walsh"
"Katherine Lee","Carrie Wood"
Sent to ma6ato@gmail.com
Removing nodegmail_gmail_run_1...
```
