title: "Hapi.js with Socket.IO - Part2: twitからTwitter Streaming APIを使う"
date: 2015-06-18 18:36:50
tags:
 - Twitter
 - SocketIO
 - Nodejs
 - twit
description: Hapi.jsとSocket.IOの練習として、前回はSocket.IOサーバー側でsetIntervalの5秒間隔でダミーのメッセージをemitするサンプルを作りました。今回はもう少し実践的にTwitter Streaming APIのPOST statuses/filterから指定したキーワードをリアルタイム検索してブラウザへ表示するサンプルにしてみます。
---

[Hapi.js](http://hapijs.com/)と[Socket.IOの練習](/2015/06/14/hapijs-socketio/)として、前回はSocket.IOサーバー側でsetIntervalの5秒間隔でダミーのメッセージをemitするサンプルを作りました。今回はもう少し実践的にTwitter Streaming APIの[POST statuses/filter](https://dev.twitter.com/streaming/reference/post/statuses/filter)から指定したキーワードをリアルタイム検索してブラウザへ表示するサンプルにしてみます。

<!-- more -->

## プロジェクト

適当なディレクトリにプロジェクトを作成します。リポジトリは[こちら](https://github.com/masato/docker-hapi-twitter)です。

```bash
$ ~/node_apps/docker-hapi-twitter
$ tree -aL 2
.
├── .dockerignore
├── .env
├── .gitignore
├── Dockerfile
├── README.md
├── app.js
├── docker-compose.yml
├── node_modules -> /dist/node_modules
├── package.json
└── templates
    └── index.html
```

### app.js

app.jsがエントリポイントです。通常のHTTPサーバーとSocket.IOサーバーの2つをラベルを付けて起動しています。Socket.IOサーバーの方はプラグインとして`twitter.js`ファイルに実装しました。

```js:~/node_apps/docker-hapi-twitter/app.js
'user strict';

var Hapi = require('hapi'),
    server = new Hapi.Server(),
    Path = require('path');
require('dotenv').load();

server.connection({ port: process.env.API_PORT, labels: ['api'] });
server.connection({ port: process.env.SOCKETIO_PORT, labels: ['twitter'] });

server.views({
    engines: {
        html: require('handlebars')
    },
    path: Path.join(__dirname, 'templates')
});

server.select('api').route({
    method: 'GET',
    path: '/',
    handler: function (request, reply) {
        reply.view('index',
                   { socketio_host: (process.env.PUBLIC_IP+':'
                                     +process.env.SOCKETIO_PORT)});
    }
});

server.register(require('./twitter'), function(err) {
    if(err) throw err;
    server.start();
});

```

### twitter.js

Twitter APIの認証情報やSocket.IOのポート番号などは[dotenv](https://www.npmjs.com/package/dotenv)が管理する`.env`ファイルに定義します。`.env.default`をリネームして使います。

```bash:~/node_apps/docker-hapi-twitter/.env
PUBLIC_IP=
SOCKETIO_PORT=8088
API_PORT=8000
TWITTER_KEY=
TWITTER_SECRET=
TWITTER_TOKEN=
TWITTER_TOKEN_SECRET=
```

twitter.jsにhapiのプラグインとしてTwitter Streaming APIの[POST statuses/filter](https://dev.twitter.com/streaming/reference/post/statuses/filter)を実装していきます。Node.jsのTwitter用パッケージはいくつかありますが、[twit](https://github.com/ttezel/twit)がきにいっています。

```js:~/node_apps/docker-hapi-twitter/twitter.js
var Twit = require('twit'),
    moment = require('moment-timezone'),
    _ = require('lodash'),
    colors = require('colors');

var T = new Twit({
    consumer_key: process.env.TWITTER_KEY,
    consumer_secret: process.env.TWITTER_SECRET,
    access_token: process.env.TWITTER_TOKEN,
    access_token_secret: process.env.TWITTER_TOKEN_SECRET
});

function formatDate(created_date){
    return moment(new Date(created_date)).tz("Asia/Tokyo").format();
}

function createUrl(screen_name,id_str){
    return 'https://twitter.com/'+screen_name+'/status/'+id_str;
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
        profile_image_url: s.user.profile_image_url,
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

function prettyPrint(tweet) {
    console.log('tweet: '.magenta.bold + tweet.text);
    console.log('by:'.green + ' @' + tweet.screen_name);
    console.log('date:'.cyan + ' ' + tweet.created_at);
    console.log('--------------------------------------------------------------------------------'.yellow);
}

exports.register = function(server,options,next) {

    var io = require('socket.io')(server.select('twitter').listener),
        track = 'babymetal',
        stream = null,
        users = [];

    io.sockets.on('connection', function(socket) {

        if(!_.includes(users, socket.id)) {
            users.push(socket.id);
        }

        var con_msg = socket.id + ' connected, now: ' + users.length; 
        console.log(con_msg);
        socket.emit('connected', con_msg);

        socket.on('disconnect', function() {
            _.pull(users, socket.id);
            var discon_msg = socket.id + ' disconnected, now: ' + users.length; 
            console.log(discon_msg);
        });

        socket.on('start stream', function() {
            console.log(socket.id + ' start twitter stream');
            if (stream === null) {
                stream =  T.stream('statuses/filter',
                                   {track: track,
                                    language: 'ja'});
            };
            stream.on('tweet', function(tweet) {
                if(users.length > 0) {
                    var data = extractData(tweet);
                    prettyPrint(data);
                    socket.emit('new tweet', data);
                } else {
                    stream.stop();
                    stream = null;
                    console.log('stop twitter stream');
                }
            });
        });
    });

    next();
};

exports.register.attributes = {
    name: 'hapi-twitter'
};
```

### 簡単な仕様

Node.jsとSocket.IO、Twitter Streaming APIの使い方は以下を参考にしてなるべく効率的なリスナーの使い方にしています。

* [Node.js, Socket.io and the Twitter Streaming API in Heroku](http://ikertxu.tumblr.com/post/56686134143/node-js-socket-io-and-the-twitter-streaming-api)
* [Using the Twitter Stream API to Visualize Tweets on Google Maps](http://blog.safe.com/2014/03/twitter-stream-api-map/)

簡単な仕様を作りました。

* ブラウザからSocket.IOの接続が来てからTwitterのストリームを開始する
* 接続しているSocket.IOのクライアントがなくなったらTwitterのストリームは閉じる



### 次回はクライアントをReagentで

キーワードは固定で`babymetal`にしています。Socket.IOサーバーができたので、次回はキーワードを画面から入力できるようなフォームを[React](http://facebook.github.io/react/)と[Reagent](http://holmsand.github.io/reagent/)で作成する予定です。最終的にはClojureとClojureScriptのフルスタックにしたいです。

### index.html

index.htmlは[Handlebars](http://handlebarsjs.com/)をテンプレートに使いました。Socket.IOの1.0から`socket.io.js`のクライアントは[cdnjs](http://cdnjs.com/libraries/socket.io)で配布されるようになりました。今回はhapiがサーバーを複数起動できることの確認のためSocket.IOサーバーのポートをHandlebarsに`{{socketio_host}}`を渡してURLを作成します。


```html:~/node_apps/docker-hapi-twitter/templates/index.html
<!DOCTYPE html>
<html lang="ja">
  <head>
    <meta charset="utf-8">
    <title>tweet</title>
    <script src="//ajax.googleapis.com/ajax/libs/jquery/2.1.4/jquery.min.js"></script>
    <script src="http://{{socketio_host}}/socket.io/socket.io.js"></script>
    <script>
      $(function() {
        if(io !== undefined) {
          var socket = io.connect("http://{{socketio_host}}");
          socket.on("connected", function(msg) {
            console.log(msg);
            socket.emit("start stream");
          });
          socket.on("new tweet", function(tweet){
            console.log(tweet);
            var img = '<img src="' + tweet.profile_image_url + '" width="48" height="48" />';
            var tweetEl = 
              '<div>'
              + '<a href="' + tweet.url + '" target="_blank">' + img + '</a>'
              + '<span style="padding-left:15px">' + tweet.name + ' @' + tweet.screen_name
              + ' - ' + tweet.created_at
              + '</span>'
              + '<p>' + tweet.text  + '</p>'
              + '</div>';
            $("#tweets").prepend(tweetEl);
          });
        }
      });
    </script>
  </head>
  <body>
    <div id="tweets"></div>
  </body>
</html>
```

## ブラウザで確認

Docker Composeでサーバーを起動します。

```bash
$ cd ~/node_apps/docker-hapi-twitter
$ docker-compose up
```

ブラウザでhapiのHTTPサーバーのURLを開くとSocket.IOクライアントがサーバーと接続を始めます。

http://xxx.xxx.xxx.xxx:8000/

ストリームで取得したtweetは少しだけ整形して`#tweets`のdivにprependします。tweetのテキスト本文と、ユーザープロファイルのイメージ、tweetのURL、ユーザー名など表示します。


```html:~/node_apps/docker-hapi-twitter/templates/index.html
var tweetEl = 
  '<div>'
  + '<a href="' + tweet.url + '" target="_blank">' + img + '</a>'
  + '<span style="padding-left:15px">' + tweet.name + ' @' + tweet.screen_name
  + ' - ' + tweet.created_at
  + '</span>'
  + '<p>' + tweet.text  + '</p>'
  + '</div>';
```

![tweet-stream.png](/2015/06/18/hapijs-socketio-twitter-streaming-api/tweet-stream.png)