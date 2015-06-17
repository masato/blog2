title: "IFTTTでYoをトリガーに任意のWebhookに通知する"
date: 2015-02-17 21:54:13
tags:
 - IFTTT
 - Express
 - Nodejs
 - Webhook
 - Yo
 - ngrok
 - RequestBin
description: IFTTTのアクションに自分で用意したWebhookに通知して欲しいときにWordPresss Channelを使うと便利です。今回はifttt-webhookのNode.js版であるexpress-ifttt-webhookを使います。AndroidからIFTTTにYoを送信するとRequestBinのエンドポイントに通知してパラメータをブラウザで確認してみます。
---

IFTTTのアクションに自分で用意したWebhookに通知して欲しいときにWordPresss Channelを使うと便利です。今回は[ifttt-webhook](https://github.com/captn3m0/ifttt-webhook)のNode.js版である[express-ifttt-webhook](https://github.com/b00giZm/express-ifttt-webhook)を使います。AndroidからIFTTTに[Yo](http://www.justyo.co/)を送信すると[RequestBin](http://requestb.in/)のエンドポイントに通知してパラメータをブラウザで確認してみます。

<!-- more -->


## Webhook

[express-ifttt-webhook](https://github.com/b00giZm/express-ifttt-webhook)はNode.jsで書かれたExpressのミドルウェアです。[google/nodejs-runtime](https://registry.hub.docker.com/u/google/nodejs-runtime/)のDockerイメージを使いExpressサーバーのコンテナを起動します。

### Webhookコンテナの用意

プロジェクトディレクトリを作成します。

``` bash
$ mkdir -p ~/docker_apps/ifttt
$ cd !$
```

package.jsonを用意して、expressとexpress-ifttt-webhookモジュールをインストールします。

```js ~/docker_apps/ifttt/package.json
{
  "name": "express-ifttt",
  "description": "express ifttt test app",
  "version": "0.0.1",
  "private": true,
  "dependencies": {
    "express": "*.*.*",
    "express-ifttt-webhook": "*.*.*"
  },
  "scripts": {"start": "node app.js"}
}
```

app.jsに簡単な処理を書きます。`json.url`には最終的にフォーワードしたいURLを指定します。今回はデバッグ用のRequestBinのURLにフォワードします。URLはこの後IFTTTのレシピに記述します。

```js ~/docker_apps/ifttt/app.js
var express = require('express')
  , webhook = require('express-ifttt-webhook');

var app = express();
app.set('port', 8080);

app.use(webhook(function(json,done){
    console.log(json);
    json.url = json.categories.string;
    done(null,json);
}));

var server = app.listen(app.get('port'), function() {
  console.log('Server listening on port', server.address().port);
});
```

Dockerfileを作成してコンテナを起動します。

``` bash
$ echo FROM google/nodejs-runtime > Dockerfile
$ docker pull google/nodejs-runtime 
$ docker build -t ifttt .
$ docker run -d --name ifttt ifttt
```

### ngrokでWebhookの公開

[ngrok](https://ngrok.com/)を使い作成したWebhookコンテナを公開します。

``` bash
$ docker pull wizardapps/ngrok:latest
$ docker run -it --rm wizardapps/ngrok:latest ngrok $(docker inspect --format="&#123;&#123; .NetworkSettings.IPAddress }}" ifttt):8080
```

公開用のngrokエンドポイントをコピーしておきます。

``` bash
...
Forwarding                    http://3b98ba34.ngrok.com -> 172.17.0.47:8080
...
```

## Yo

[Yo](http://www.justyo.co/)はYoと通知するだけのシンプルなコミュニケーションツールです。IFTTTのトリガーにYoを使ってみます。AndroidにYoアプリをインストールして、+ボタンを押し`IFTTT`をユーザーに追加しておきます。

## IFTTT

### Yoチャンネルのアクティベート

チャンネルからYoを検索してアクティベートします。

![yo-activate.png](/2015/02/17/express-ifttt-webhook-requestbin/yo-activate.png)

### WordPressチャンネルのアクティベート

チャネルからWordPressを検索します。テスト用なので今回は認証を行いませんが、UsernameとPasswordはWordPress Channelで必須項目のため適当に入力します。

* Blog URL: http://3b98ba34.ngrok.com
* Username: username
* Password: password

![wordpress-activate.png](/2015/02/17/express-ifttt-webhook-requestbin/wordpress-activate.png)

### レシピの作成

最初に`This`をクリックしてYoのトリガーを選択します。

![choose-trigger.png](/2015/02/17/express-ifttt-webhook-requestbin/choose-trigger.png)

次に`That`をクリックしてWordPress Channelをアクションに選択します。

![choose-action.png](/2015/02/17/express-ifttt-webhook-requestbin/choose-action.png)

`Create post`のアクションを選択します。フィールドは任意に設定できますが、`Categories`フィールドにWebfookしたい最終的なフォワード先のURLを指定します。今回は[RequestBin](http://requestb.in/)で作成したURLを入力します。`Create Action`ボタンを押してアクションのを完了します。

* Categories: http://requestb.in/pb4l5spb

![categories-field.png](/2015/02/17/express-ifttt-webhook-requestbin/categories-field.png)

最後に`Create Recipe`ボタンをクリックしてレシピのアクティベートを行います。

![activate-recipe.png](/2015/02/17/express-ifttt-webhook-requestbin/activate-recipe.png)

## AndroidのYoアプリからテスト

AndroidにインストールしたYoアプリを起動します。追加したユーザーの`IFTTT`をタップします。「Yo送信完了!」と表示されれば成功です。

![ifttt.png](/2015/02/17/express-ifttt-webhook-requestbin/ifttt.png)

RequestBinで先ほど作成したURLを開くとIFTTTからPOSTされたパラメーターを確認することができました。

![requestbin.png](/2015/02/17/express-ifttt-webhook-requestbin/requestbin.png)

以下がIFTTTから通知されるJSONデータの中身です。コールバックではJSONオブジェクトになっています。

``` json
{ username: 'username',
  password: 'password',
  title: 'Yo',
  description: 'hello',
  categories: { string: 'http://requestb.in/pb4l5spb' },
  tags: [ { string: 'IFTTT' }, { string: 'Yo' } ],
  post_status: 'publish' }
```

今回はテスト的に通知を受けただけですが、次回はExpressの中でMQTTにブリッジしてもう少し汎用的に使えるようにしてみます。