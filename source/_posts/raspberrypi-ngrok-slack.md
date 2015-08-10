title: "Raspberry Piをngrokで公開する - Part2: ngrokのURLをSlackに通知する"
date: 2015-08-05 13:52:42
categories:
 - IoT
tags:
 - RaspberryPi
 - ngrok
 - pm2
 - Nodejs
 - Slack
description: 前回Node.jsのngrokラッパーを使いRaspberry Piで動作しているExpressをインターネットに公開しました。しかし生成されたランダムなngrokのURLはRaspberry Piにログインしてpm2のログから取得する必要がありました。次はこのURLをSlackに投稿してリモートから確認できるようにします。
---

[前回](/2015/08/04/raspberrypi-ngrok/)Node.jsのngrokラッパーを使いRaspberry Piで動作しているExpressをインターネットに公開しました。しかし生成されたランダムなngrokのURLはRaspberry Piにログインしてpm2のログから取得する必要がありました。次はこのURLをSlackに投稿してリモートから確認できるようにします。

<!-- more -->

## Slack

SlackのConfigure Integrations(https://{teamdomain}.slack.com/services/new)のページから、Incoming WebHooks(https://{teamdomain}.slack.com/services/new/incoming-webhook)を開きます。適当なチャンネルを選択して`Add Incoming WebHooks Integration`を押します。


![incoming-webhook.png](/2015/08/05/raspberrypi-ngrok-slack/incoming-webhook.png)


作成されたWebhook URLはあとで.envファイルの環境変数に設定します。

![incoming-webhook-url.png](/2015/08/05/raspberrypi-ngrok-slack/incoming-webhook-url.png)


## プロジェクト

### node-slack

前回作成した[プロジェクト](https://github.com/masato/node-hello-ngrok-pm2)にNode.jsのSlackモジュールの一つである、[node-slack](https://www.npmjs.com/package/node-slack)をインストールします。

```bash
$ cd ~/node_apps/node-hello-ngrok-pm2
$ npm install node-slack --save
```

package.jsonのdependenciesにnode-slackが追加されました。

```json ~/node_apps/node-hello-ngrok-pm2/package.json
{
  "name": "node-hello-ngrok-pm2",
  "description": "node-hello-ngrok-pm2",
  "version": "0.0.1",
  "private": true,
  "scripts": {
    "start": "node app.js"
  },
  "dependencies": {
    "dotenv": "^1.2.0",
    "express": "^4.13.3",
    "ngrok": "^0.1.99",
    "node-slack": "0.0.7"
  }
}
```

dotenvのひな形をリネームします。

```bash
$ mv .env.default .env
```

先ほど取得したSlackのWebhook URLを追加します。

```bash ~/node_apps/node-hello-ngrok-pm2/.env
EXPRESS_PORT=3000
SLACK_WEBHOOK=https://hooks.slack.com/services/xxxx
```

### app.js

app.jsにnode-slackのコードを追加します。`ngrok.connect`のコールバック内で取得したngrokのURLを、Slackのメッセージとして`#general`チャンネルに投稿します。

```js ~/node_apps/node-hello-ngrok-pm2/app.js
require('dotenv').load();
var ngrok = require('ngrok'),
    express = require('express'),
    Slack = require('node-slack');

var slack = new Slack(process.env.SLACK_WEBHOOK);
var app = express();

app.get('/', function(req, res) {
    res.send('Hello World!');
});

var server = app.listen(process.env.EXPRESS_PORT, function() {
    var host = server.address().address,
        port = server.address().port;

    console.log('listening at http://%s:%s', host, port);
    ngrok.connect({
        port: port
    },
    function (err, url) {
        console.log(url);
        var message = 'app url: ' + url;
        slack.send({
            text: message,
            channel: '#general',
            username: 'raspi',
            icon_emoji: ":ghost:"
        });
    });
});
```



## 確認

pm2からアプリをリスタートします。

```bash
$ pm2 restart app
[PM2] restartProcessId process id 0
┌──────────┬────┬──────┬──────┬────────┬─────────┬────────┬────────────┬──────────┐
│ App name │ id │ mode │ pid  │ status │ restart │ uptime │ memory     │ watching │
├──────────┼────┼──────┼──────┼────────┼─────────┼────────┼────────────┼──────────┤
│ app      │ 0  │ fork │ 4401 │ online │ 7       │ 0s     │ 8.602 MB   │ disabled │
└──────────┴────┴──────┴──────┴────────┴─────────┴────────┴────────────┴──────────┘
 Use `pm2 show <id|name>` to get more details about an app
```

リスタート後、SlackにngrokのURLが投稿されました。


![slack-ngrok-url.png](/2015/08/05/raspberrypi-ngrok-slack/slack-ngrok-url.png)

Slackに投稿されたリンクをクリックすると、Raspberry Piで起動しているExpressの画面が表示されます。

![raspi-hello-ngrok.png](/2015/08/05/raspberrypi-ngrok-slack/raspi-hello-ngrok.png)
