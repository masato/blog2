title: "myThingsをはじめよう - Part6: 「IDCF」のトリガーからHTTPで「Gmail」のアクションを実行する"
date: 2015-09-01 11:52:59
categories:
 - IoT
tags:
 - IoT
 - Meshblu
 - myThings
 - IDCFクラウド
description: 「IDCF」チャンネルを使うと、Raspberry Piと環境センサーが計測した気温が27度を超えたときなど閾値を監視して、myThingsのトリガーを作成することができます。前回はアクションを作成するサンプルでしたが、今回はトリガーを作成するサンプルになります。
---

「IDCF」チャンネルを使うと、Raspberry Piと環境センサーが計測した気温が27度を超えたときなど閾値を監視して、myThingsのトリガーを作成することができます。[前回](/2015/08/31/mythings-idcfchannel-action-http/)はアクションを作成するサンプルでしたが、今回はトリガーを作成するサンプルになります。

<!-- more -->

## myThingsアプリ

### 「Gmail」チャンネルの承認

myThingsのトリガーに「Gmail」チャンネルを使います。myThingsアプリを開きチャンネル一覧から選択して認証します。

![gmail-channel.png](/2015/09/01/mythings-idcfchannel-trigger-http/gmail-channel.png)


### 組み合わせの作成

ホーム画面から右上のプラスボタンをタップします。

![mythings-home.png](/2015/09/01/mythings-idcfchannel-trigger-http/mythings-home.png)

組み合わせ作成画面ではトリガーとアクションのチャンネルをプラスボタンをタップして選択します。

![recipe.png](/2015/09/01/mythings-idcfchannel-trigger-http/recipe.png)


### 「IDCF」トリガー

トリガーのプラスボタンをタップして、チャンネル一覧から「IDCF」をトリガーに選択します。「IDCF」チャンネルの認証をまだしていない場合は、[myThingsをはじめよう - Part4: 「IDCF」チャンネルを認証する]/2015/08/30/mythings-idcfchannel-activation/)の手順で認証します。

![idcf-triger.png](/2015/09/01/mythings-idcfchannel-trigger-http/idcf-trigger.png)


「条件を満たしたら」のリストボックスから、`trigger-1`を選択します。

![idcf-trigger-selected.png](/2015/09/01/mythings-idcfchannel-trigger-http/idcf-trigger-selected.png)

OKボタンをタップすると組み合わせの作成画面に戻ります。

![recipe-trigger.png](/2015/09/01/mythings-idcfchannel-trigger-http/recipe-trigger.png)


### 「Gmail」アクション

アクションのプラスボタンをタップして、チャンネル一覧から「Gmail」をアクションに選択します。

![gmail-action.png](/2015/09/01/mythings-idcfchannel-trigger-http/gmail-action.png)


アクションの選択画面で「メールを送信する」をタップします。

![gmail-action-mail.png](/2015/09/01/mythings-idcfchannel-trigger-http/gmail-action-mail.png)


送信先メールアドレス、件名、本文はすべて必須項目です。

![gmail-action-detail.png](/2015/09/01/mythings-idcfchannel-trigger-http/gmail-action-detail.png)

アクションの詳細設定が終わったらOKボタンをタップすると、組み合わせの作成画面に戻ります。


とりあえず「実行タイミング」と「曜日指定」はデフォルトのまま、「作成」ボタンをタップして終了します。

![gmail-action-created.png](/2015/09/01/mythings-idcfchannel-trigger-http/gmail-action-created.png)

組み合わせの作成が完了して組み合わせ一覧画面に戻ります。


## 「IDCF」チャンネルサーバー

### トリガーのURLとデータの保存

myThingsサーバーは一定間隔でメッセージの受信を確認しています。「IDCF」チャンネルをトリガーに使う場合は、myThingsサーバーが非同期でメッセージを取得できるようにデータを保存しておく必要があります。

HTTPを使う場合は、`/data/{triggerのuuid}`にメッセージをPOSTすると「IDCF」チャンネルのデータベースにデータが保存されます。myThingsサーバーは新しいデータが取得できると、トリガー条件を満たしてアクションを実行します。


### trigger-1デバイスの確認

最初に「IDCF」チャンネルサーバーをインストールしたIDCFクラウドの仮想マシンにログインします。myThingsアプリのアクションに選択した`trigger-1`のuuidとtokenを確認します。

```bash
$ cd ~/iot_apps/meshblu-compose
$ docker-compose run --rm iotutil show -- -k trigger-1

> iotutil@0.0.1 start /app
> node app.js "show" "-k" "trigger-1"

┌───────────┬──────────┬──────────────────────────────────────┐
│ keyword   │ token    │ uuid                                 │
├───────────┼──────────┼──────────────────────────────────────┤
│ trigger-1 │ d74ebedf │ 21c83792-b25e-4ae7-a627-714af57a1a4b │
└───────────┴──────────┴──────────────────────────────────────┘
```

### curlコマンドのトリガー

確認したデバイス情報を使ってcurlコマンドを実行します。URLの`data`の後ろに`trigger-1`のuuidを指定します。また、HTTPヘッダの認証情報にmyThingsアプリのトリガーに指定した`trigger-1`のuuidとtokenを指定します。

* URL: 「IDCF」チャンネルに割り当てたIPアドレス + '/data/' + uuid
* データ: 任意(現在このデータをアクションに渡す事はできません)
* meshblu_auth_uuid: trigger-1のuuid
* meshblu_auth_token: trigger-1のtoken

```bash
$ curl -X POST \
  "http://210.140.162.58/data/21c83792-b25e-4ae7-a627-714af57a1a4b" \
  -d 'temperature=over' \
  --header "meshblu_auth_uuid: 21c83792-b25e-4ae7-a627-714af57a1a4b" \
  --header "meshblu_auth_token: d74ebedf"
```

### Pythonのトリガー

スイッチサイエンスから発売されている[myThingsをはじめようキット](https://www.switch-science.com/catalog/2366/)にも同梱されている[BME280](https://www.switch-science.com/catalog/2236/)の環境センサーを使ったサンプルコードです。リポジトリは[こちら](https://github.com/IDCFChannel/bme280-meshblu-py/blob/master/bme280_data.py)です。計測した気温がconfig.pyに定義した閾値の26度を超えたら、「IDCF」チャンネルサーバーにHTTP POSTします。

```python bme280_data.py
#!/usr/bin/python
# -*- coding: utf-8 -*-
from time import sleep
import json
import bme280
import requests
from config import conf

def sensing():
    return bme280.readData()

def main():
    # HTTP
    url = "http://{0}/data/{1}".format(conf["IDCF_CHANNEL_URL"],
                                       conf["TRIGGER_1_UUID"])
    headers = {
        "meshblu_auth_uuid": conf["TRIGGER_1_UUID"],
        "meshblu_auth_token": conf["TRIGGER_1_TOKEN"]
    };
    payload = {"trigger":"on"}

    while True:
        retval = sensing()
        if retval:
             if float(retval["temperature"]) > conf["THRESHOLD"]:
                 r = requests.post(url, headers=headers, data=payload)
        sleep(5)

if __name__ == '__main__':
    main()
```

設定ファイルには、「IDCF」チャンネルサーバーのパブリックIPアドレス、`trigger-1`のuuidとtokenも指定します。

```python config.py
conf = {
     "IDCF_CHANNEL_URL": "210.140.162.58",
     "TRIGGER_1_UUID": "21c83792-b25e-4ae7-a627-714af57a1a4b",
     "TRIGGER_1_TOKEN": "d74ebedf",
     "THRESHOLD": 27.0
}
```

依存関係にある[requests](https://pypi.python.org/pypi/requests)パッケージをインストールします。

```bash
$ sudo pip install requests
```

プログラムの実行にはroot権限が必要です。

```bash
$ chmod +x bme280_data.py
$ sudo ./bme280_data.py
```

### データの確認

IDCFクラウドの仮想マシンにログインして「IDCF」チャンネルをインストールしたディレクトリに移動してMongoDBのレコードを確認してみます。

```bash
$ cd ~/iot_apps/meshblu-compose
$ docker exec -it meshblucompose_mongo_1 mongo skynet
> db.data.find().sort({ $natural: -1 }).limit(1)
{ "_id" : ObjectId("55e55ba8a9f3a50f0013bcee"), "temperature" : "over", "ipAddress" : "210.168.36.155", "uuid" : "21c83792-b25e-4ae7-a627-714af57a1a4b", "timestamp" : "2015-09-01T08:02:48.465Z" }
>
```

## 手動実行でテスト

myThingsアプリから組み合わせ一覧を開きます。

![recipe-list.png](/2015/09/01/mythings-idcfchannel-trigger-http/recipe-list.png)

IDCFとGmailの組み合わせをタップして詳細画面に移動します。

![gmail-action-created.png](/2015/09/01/mythings-idcfchannel-trigger-http/gmail-created.png)

「手動実行」ボタンをタップすると、デフォルトで15分間隔の実行タイミングを待たなくても、すぐに組み合わせを実行することができます。

![recipe-executed.png](/2015/09/01/mythings-idcfchannel-trigger-http/recipe-executed.png)

「タイムライン」画面に移動すると、手動実行の履歴が表示されています。

![recipe-history.png](/2015/09/01/mythings-idcfchannel-trigger-http/recipe-history.png)

Gmailアプリの受信トレイを開くと、myThingsのアクションからメールが届きました。

![gmail-received.png](/2015/09/01/mythings-idcfchannel-trigger-http/gmail-received.png)
