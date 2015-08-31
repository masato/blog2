title: "myThingsをはじめよう - Part5: 「天気・災害」のトリガーを「IDCF」のアクションがHTTPで受信する"
date: 2015-08-31 11:52:59
categories:
 - IoT
tags:
 - IoT
 - Meshblu
 - myThings
 - IDCFクラウド
description: 「IDCF」チャンネルは主に自作デバイスをmyThingsと接続するためのチャンネルですが、HTTPクライアントが動作する環境ならどこでも使うことができます。例としてOSXのターミナル上でcurlコマンドを「IDCF」チャンネルのアクションとして使います。「天気・災害」チャンネルをトリガーにして、「IDCF」チャンネルのアクションを実行する組み合わせを書いてみます。
---

[「IDCF」チャンネル](http://www.idcf.jp/cloud/iot/)は主に自作デバイスを[myThings](http://mythings.yahoo.co.jp/)と接続するためのチャンネルですが、HTTPクライアントが動作する環境ならどこでも使うことができます。例としてOSXのターミナル上でcurlコマンドを「IDCF」チャンネルのアクションとして使います。「天気・災害」チャンネルをトリガーにして、「IDCF」チャンネルのアクションを実行する組み合わせを書いてみます。

<!-- more -->


## myThingsアプリ

### 「天気・災害」チャンネルの承認

トリガーに「天気・災害」チャンネルを使います。myThingsアプリを開きチャンネル一覧から選択して認証します。

![weather-channel.png](/2015/08/31/mythings-idcfchannel-action-http/weather-channel.png)

### 組み合わせの作成

ホーム画面から右上のプラスボタンをタップします。

![mythings-home.png](/2015/08/31/mythings-idcfchannel-action-http/mythings-home.png)

組み合わせ作成画面ではトリガーとアクションのチャンネルをプラスボタンをタップして選択します。

![idcf-channel-recipe.png](/2015/08/31/mythings-idcfchannel-action-http/idcf-channel-recipe.png)


### 「天気・災害」トリガー

トリガーのプラスボタンをタップして、チャンネル一覧から「天気・災害」をトリガーに選択します。アクション詳細設定画面で「今日の天気予報」をタップします。


![weather-channel-today.png](/2015/08/31/mythings-idcfchannel-action-http/weather-channel-today.png)


天気予報を知りたい「地域」を選択します。今回は「新宿」にしました。

![weather-channel-trigger-selected.png](/2015/08/31/mythings-idcfchannel-action-http/weather-channel-trigger-selected.png)


OKボタンをタップすると組み合わせの作成画面に戻ります。


### 「IDCF」アクション

アクションのプラスボタンをタップして、チャンネル一覧から「IDCF」をアクションに選択します。アクション詳細設定画面で「通知する」をタップします。

![idcf-channel-notify.png](/2015/08/31/mythings-idcfchannel-action-http/idcf-channel-notify.png)

「アクションを選択する」のリストボックスから、`action-1`を選択します。

![idcf-channel-action-1.png](/2015/08/31/mythings-idcfchannel-action-http/idcf-channel-action-1.png)

「メッセージ」のテキストエリアをタップすると、予めセットされた「候補」を選ぶことができます。ここから以下のようにカンマ区切りでいくつか選択します。中括弧のプレイスホルダは実際の天気情報に置換されて、アクションがメッセージとして受信することができます。

```
{{地点名}},{{天候の状態}},{{最高気温}},{{天候状態の画像のURL}}
```

![idcf-channel-message.png](/2015/08/31/mythings-idcfchannel-action-http/idcf-channel-message.png)


アクションの詳細設定が終わったらOKボタンをタップすると、組み合わせの作成画面に戻ります。

![idcf-channel-action-selected.png](/2015/08/31/mythings-idcfchannel-action-http/idcf-channel-action-selected.png)

とりあえず「実行タイミング」と「曜日指定」はデフォルトのまま、作成ボタンをタップして終了します。

![idcf-channel-action-create.png](/2015/08/31/mythings-idcfchannel-action-http/idcf-channel-action-create.png)

組み合わせの作成が完了して一覧画面に戻ります。

![idcf-channel-action-created.png](/2015/08/31/mythings-idcfchannel-action-http/idcf-channel-action-created.png)


## 「IDCF」チャンネルサーバー

### action-1デバイスの確認

最初に「IDCF」チャンネルサーバーをインストールしたIDCFクラウドの仮想マシンにログインします。myThingsアプリのアクションに選択した`action-1`のuuidとtokenを確認します。

```bash
$ cd ~/iot_apps/meshblu-compose
$ docker-compose run --rm iotutil show -- -k action-1

> iotutil@0.0.1 start /app
> node app.js "show" "-k" "action-1"

┌──────────┬──────────┬──────────────────────────────────────┐
│ keyword  │ token    │ uuid                                 │
├──────────┼──────────┼──────────────────────────────────────┤
│ action-1 │ 8a781e76 │ 3a78814a-6879-4543-bacf-9a206cd951a6 │
└──────────┴──────────┴──────────────────────────────────────┘
```

### curlコマンドのアクション

確認したデバイス情報を使ってcurlコマンドを実行します。HTTPヘッダの認証情報にmyThingsアプリのアクションに指定した`action-1`のuuidとtokenを指定することで、myThingsのトリガーからのメッセージを`subscribe`します。Meshbluサーバーはセッションを閉じないのでcurlコマンドは待ち受け状態になります。

* URL: 「IDCF」チャンネルに割り当てたIPアドレス + '/subscribe'
* meshblu_auth_uuid: action-1のuuid
* meshblu_auth_token: action-1のtoken

```bash
$ curl -X GET \
  "http://210.140.162.58/subscribe" \
  --header "meshblu_auth_uuid: 3a78814a-6879-4543-bacf-9a206cd951a6" \
  --header "meshblu_auth_token: 8a781e76"
```

このcurlコマンドはしばらくするとタイムアウトします。タイムアウトする前にmyThingsアプリから「手動実行」でテストします。

## 手動実行でテスト

myThingsアプリから組み合わせ一覧を開きます。

![idcf-channel-recipe-list.png](/2015/08/31/mythings-idcfchannel-action-http/idcf-channel-recipe-list.png)

先ほど作成したばかりの組み合わせをタップして詳細画面に移動します。

![idcf-channel-recipe-detail.png](/2015/08/31/mythings-idcfchannel-action-http/idcf-channel-recipe-detail.png)

「手動実行」ボタンをタップすると、デフォルトで15分間隔の実行タイミングを待たなくても、すぐに組み合わせを実行することができます。

![idcf-channel-recipe-executed.png](/2015/08/31/mythings-idcfchannel-action-http/idcf-channel-recipe-executed.png)


### curlコマンドのアクション結果

curlコマンドで`action-1`をsubscribeしているシェルに戻ります。「IDCF」チャンネルのアクションの結果として以下のようなJSONメッセージを「天気・災害」チャンネルのトリガーから受信することができました。

```json
{"devices":["3a78814a-6879-4543-bacf-9a206cd951a6"],"payload":"東京（東京）,曇り,24,http://i.yimg.jp/images/weather/general/forecast/clouds.gif","fromUuid":"12a95246-f31e-4f0c-9678-2ae755e57b98"}
```
