title: "MQTTでGmailからSlackへ転送するためのCompose"
date: 2015-05-01 16:59:29
categories:
 - IoT
tags:
 - Meshblu
 - MQTT
 - Slack
 - DockerCompose
 - Nodejs
description: 特定のキーワードを含む件名のメールがGmailに届いたらSlackに転送する、Hubotみたいな処理をMQTTブローカーを使って試してみます。publishとsubscrbeはそれぞれDockerコンテナで実装します。MQTTがしゃべれるコネクテッドデバイスならtopicをsubscribeできるので、どちらもメッセージングのブリッジに使えそうです。
---

特定のキーワードを含む件名のメールがGmailに届いたらSlackに転送する、Hubotみたいな処理をMQTTブローカーを使って試してみます。publishとsubscrbeはそれぞれDockerコンテナで実装します。MQTTがしゃべれるコネクテッドデバイスならtopicをsubscribeできるので、どちらもメッセージングのブリッジに使えそうです。

<!-- more -->

## プロジェクトの作成

Docker Composeでサービスを構成するディレクトリを作成します。config.jsonにはユーザー名やパスワートがあります。ビルド用のシェルスクリプトを用意したほうが良さそうですが、Composeを使わなくてもディレクトリでもDockerイメージをビルドできるようにしました。

``` bash
$ cd ~/node_apps/gmail-slack/
$ tree
.
├── docker-compose.yml
├── gmail-pub
│   ├── Dockerfile
│   ├── app.js
│   ├── config.json
│   └── package.json
└── slack-sub
    ├── Dockerfile
    ├── app.js
    ├── config.json
    └── package.json
```

Composeでサービスをビルドすると一度に実行できるので便利です。

```yaml ~/node_apps/gmail-slack/docker-compose.yml
gmail:
  build: ./gmail-pub
  restart: always
slack:
  build: ./slack-sub
  restart: always
```

## MeshbluのMQTTブローカー

MQTTブローカーにはMeshbluを使います。MQTTのtopic/username/passwordに相当するデバイスとTokenを作成します。tokenは適当にランダムの文字列を生成しました。

``` bash
$ curl -X POST \
  http://localhost/devices \
  -d "name=gmail-action&uuid=gmail-action&token=409qGstB"
```

## Gmail > MQTT

最初のサービスはGmailの新着メールを監視して、キーワードを含む件名のメール本文をMQTTブローカーにpublishするDockerコンテナです。Node.jsのONBUILをベースイメージにします。

```bash ~/node_apps/gmail-slack/gmail-pub/Dockerfile
FROM node:0.12-onbuild
```

`config.json`はMQTTブローカーの接続情報と、Gmailの認証情報です。今回のテスト用にGmailのアカウントは新規に取得して以下の設定をします。

* アカウント設定からログイン項目内の安全性の低いアプリのアクセスをオン
* IMAPの設定を有効

`gmailTitle`にはSlackにメール本文を転送したい場合、件名に含めるキーワードを指定します。

```json ~/node_apps/gmail-slack/gmail-pub/config.json
{"mqttHost": "xxx",
 "mqttUser": "gmail-action",
 "mqttPass": "409qGstB",
 "gmailTitle": "[slackme]",
 "gmailUser":"xxx@gmail.com",
 "gmailPass":"xxx"}
```

`package.json`に必要なパッケージを書きます。日本語のメールを扱う場合はiconvはデフォルトでiconvの依存関係に入っていないので明示的に追加します。

```json ~/node_apps/gmail-slack/gmail-pub/package.json
{
  "name": "gmail-pub",
  "description": "gmail-pub app",
  "version": "0.0.1",
  "private": true,
  "dependencies": {
    "iconv": "^2.1.7",
    "iconv": "^1.1.59",
    "mailparser": "^0.5.1",
    "mqtt":"^1.1.3"
  },
  "scripts": {
    "start": "node app.js"
  }
}
```

GmailのINBOXを監視するパッケージは[inbox](https://github.com/pipedrive/inbox)を使いました。[mailparser](https://github.com/andris9/mailparser)でメールの本文をストリームから文字列にします。また、`regexpEscape`関数はメール転送のマーカーとして件名に含まれるキーワードに`[slackme]`とカギ括弧を使ってしまったのでバックスラッシュでエスケープさせています。publishするメッセージフォーマットはMesubluの仕様です。topicは`message`固定、`devices`にメッセージ送信先を指定します。`payload`が実際に送信したいメッセージ本文になります。

```js ~/node_apps/gmail-slack/gmail-pub/app.js
var inbox = require('inbox'),
    MailParser = require("mailparser").MailParser,
    mqtt = require('mqtt'),
    config = require('./config.json'),
    opts = {
        host: config.mqttHost,
        username: config.mqttUser,
        password: config.mqttPass
    },
    mqttClient = mqtt.connect(opts);

var imapClient = inbox.createConnection(false, "imap.gmail.com", {
    secureConnection: true,
    auth: {
        user: config.gmailUser,
        pass: config.gmailPass
    }
});

imapClient.connect();

imapClient.on("connect", function() {
    imapClient.openMailbox("INBOX", function(error, info) {
        if(error) throw error;
        console.log("Successfully connected to Gmail");
    });
});

function regexpEscape(s) {
    return s.replace(/[-\/\\^$*+?.()|[\]{}]/g, '\\$&');
}

imapClient.on("new", function(message) {
    var exp = new RegExp(regexpEscape(config.gmailTitle))
    if (message.title.match(exp)) {
        var stream = imapClient.createMessageStream(message.UID);
        var mailParser = new MailParser();
        mailParser.on("end",function(mail) {
            console.log(mail.text);
            var payload = {"devices":config.mqttUser,
                           "payload":mail.text};
            mqttClient.publish('message', JSON.stringify(payload));
        })
        stream.pipe(mailParser)
    }
});
```

## MQTT > Slack

2つ目のサービスはMQTTブローカーからGmailの本文をsubscribeしてSlackに転送するDockerコンテナです。Dockerfileは同じです。

```bash ~/node_apps/gmail-slack/slack-sub/Dockerfile
FROM node:0.12-onbuild
```

最初のコンテナを重複しますがMQTTブローカーとSlackの認証情報を`config.json`に定義します。Slackのチャンネルは`#gmail`を用意します。

```json ~/node_apps/gmail-slack/slack-sub/config.json
{"mqttHost": "xxx",
 "mqttUser": "gmail-action",
 "mqttPass": "409qGstB",
 "slackUser": "xxx",
 "slackHookUrl": "https://hooks.slack.com/services/xxx",
 "slackChannel":"#gmail",
 "slackIcon": ":ghost:"
}
```

SlackのNode.jsクライアントは[npmjs.org](https://www.npmjs.com/search?q=slackにたくさんありますが、[slack-node](https://github.com/clonn/slack-node-sdk)が使いやすそうです。

```json ~/node_apps/gmail-slack/slack-sub/package.json
{
  "name": "slack-sub",
  "description": "slack-sub app",
  "version": "0.0.1",
  "private": true,
  "dependencies": {
    "slack-node": "^0.1.0",
    "mqtt":"^1.1.3"
  },
  "scripts": {
    "start": "node app.js"
  }
}
```

MQTTブローカーからsubscribeしてGmailからpublishされたメッセージを取得します。Slackから[Incoming Webhooks](https://api.slack.com/incoming-webhooks)を作成してhook URLを取得しておきます。

```js ~/node_apps/gmail-slack/slack-sub/app.js
var Slack = require('slack-node'),
    slack = new Slack(),
    mqtt = require('mqtt'),
    config = require('./config.json'),
    opts = {
        host: config.mqttHost,
        username: config.mqttUser,
        password: config.mqttPass
    },
    mqttClient = mqtt.connect(opts);

mqttClient.subscribe(config.mqttUser);
mqttClient.on("message", function(topic, message) {

    var json = JSON.parse(message);
    var text = json.data.payload;

    slack.setWebhook(config.slackHookUrl);
    slack.webhook({
        channel: config.slackChannel,
        username: config.slackUser,
        icon_emoji: config.slackIcon,
        text: text
    },function(err, response) {
        console.log(response);
    });
});
```

## 実行

初回に`docker-compose up`するとビルドしてからサービスを起動します。イメージの修正が必要な場合2回目以降は`docker-component build`を実行します。

``` bash
$ cd ~/node_apps/gmail-slack
$ docker-compose up
...
slack_1 |
slack_1 | > slack-sub@0.0.1 start /usr/src/app
slack_1 | > node app.js
slack_1 |
gmail_1 |
gmail_1 | > gmail-pub@0.0.1 start /usr/src/app
gmail_1 | > node app.js
gmail_1 |
gmail_1 | Successfully connected to Gmail
```

config.jsonに登録したGmailアカウントにメールを送信します。

* 宛先:　テスト用のGmailアカウント
* 件名: [slackme] テスト
* 本文: てすとめーる

Docker Composeを使うとforemanみたいに複数のサービスの標準出力を同時に確認できるので便利です。色分けされて見やすいです。

``` bash 
...
gmail_1 | Gmail message: てすとめーる
gmail_1 |
slack_1 | { status: 'ok',
slack_1 |   statusCode: 200,
...
slack_1 |   response: 'ok' }
```

Slackにメール本文が転送されました。Android Wearを使っていると振動して通知してくれます。

![slack-ghost.png](/2015/05/01/meshblu-mqtt-gmail-slack/slack-ghost.png)
