title: "myThingsをはじめよう - Part7: トリガーの閾値監視をRaspberry Piで行う"
date: 2015-09-16 14:40:21
categories:
 - IoT
tags:
 - IoT
 - RaspberryPi
 - Meshblu
 - myThings
 - BME280
 - IDCFクラウド
description: myThingsの組合せとしてIDCFをトリガーに使う場合、何かしらの条件を設定してその条件を満たしているか判定する必要があります。大きく分けてRaspberry Piなどのコネクテッドデバイス側か、IDCFチャンネルサーバーがあるクラウド側でプログラムを実行します。今回は計測したセンサーデータが閾値を超えた場合に、トリガー発火条件を満たしているか判定するコードをRasbperry Pi側で実装してみます。
---

myThingsの組合せとしてIDCFをトリガーに使う場合、何かしらの条件を設定してその条件を満たしているか判定する必要があります。大きく分けてRaspberry Piなどのコネクテッドデバイス側か、IDCFチャンネルサーバーがあるクラウド側でプログラムを実行します。今回は計測したセンサーデータが閾値を超えた場合に、トリガー発火条件を満たしているか判定するコードをRasbperry Pi側で実装してみます。

<!-- more -->

## はじめに

### 作業の流れ

以下の手順で作業を進めていきます。

1. Raspberry PiでBME280センサーデータを取得する
1. Raspberry Piで閾値監視をする
1. Raspberry PiからIDCFチャンネルサーバーにHTTP POST `/data/{triggerのuuid}`する
1. myThingsアプリから組合せを「手動実行」する
1. Gmailにメールが届く

### Raspberry PiとBME280環境センサー設定

あらかじめ[Part2](/2015/07/16/raspberrypi-2-headless-install-2/)の手順でRaspberry Piと[BME280環境センサモジュール](https://www.switch-science.com/catalog/2236/)のセットアップを行います。

すでにサンプルコードをダウンロード済の場合は`git clone`したディレクトリに移動して最新のコードを取得します。

```bash
$ cd ~/python_apps/bme280-meshblu-pybme280-meshblu-py
$ git pull
```

あたらしくダウンロードする場合はディレクトリを作成して`git clone`します。

```bash
$ mkdir -p ~/python_apps
$ cd ~/python_apps
$ git clone https://github.com/IDCFChannel/bme280-meshblu-py.git
$ cd bme280-meshblu-py
```

[bme280_sample.py](https://github.com/IDCFChannel/bme280-meshblu-py/blob/master/bme280_sample.py)のPythonプログラムを実行してセンサーからデータを取得できることを確認します。

```bash
$ sudo python bme280_sample.py
temp : 27.06  ℃
pressure : 1006.96 hPa
hum :  57.78 ％
```

### IDCFチャンネルサーバーでtrigger-1の確認

IDCFクラウドの仮想マシンにログインします。IDCFチャンネルサーバーが起動しているディレクトリに移動して`list`コマンドを実行します。使用する`trigger-1`のtokenとuuidを確認します。

```bash
$ cd ~/iot_apps/meshblu-compose/
$ docker-compose run --rm iotutil list
...
┌───────────┬──────────┬──────────────────────────────────────┐
│ keyword   │ token    │ uuid                                 │
├───────────┼──────────┼──────────────────────────────────────┤
│ trigger-1 │ d74ebedf │ 21c83792-b25e-4ae7-a627-714af57a1a4b │
├───────────┼──────────┼──────────────────────────────────────┤
...
```

### Raspberry Piでコードの準備

Rasbpberry Piにログインしてconfig.pyを編集します。IDCFチャンネルサーバーで確認した`trigger-1`の値や、IDCFチャンネルサーバーのIPアドレスを設定します。actionのuuidなどは今回は使用しません。`THRESHOLD`は閾値監視で使う気温の閾値になります。

```python config.py
conf = {
     "IDCF_CHANNEL_URL": "210.140.162.58",
     "TRIGGER_1_UUID": "21c83792-b25e-4ae7-a627-714af57a1a4b",
     "TRIGGER_1_TOKEN": "d74ebedf",
     "ACTION_1_UUID": "",
     "ACTION_2_UUID": "",
     "THRESHOLD": 27.0
}
```

### myThingsアプリで組合せの作成

IDCFをトリガーに、Gmailをアクションにして組合せを作成します。作り方は[Part6の「Gmail」のアクションを実行する](/2015/09/01/mythings-idcfchannel-trigger-http/)と同じです。

![gmail-action-created.png](/2015/09/16/mythings-idcfchannel-trigger-edge/gmail-action-created.png)


## Raspberry Pi側で閾値監視

Raspberry Pi上でセンサーデータを取得するプログラムの中で閾値監視も同時に行うパターンです。クラウド側に追加のプログラムが不要なため、単純な閾値監視やテストに向いています。ただしRaspberry Piが複数になった場合に、閾値監視の条件を変更した後のプログラムのデプロイが煩雑になります。


### Raspberry Pi側のプログラム

最初にプログラムで利用する[requests](http://www.python-requests.org/en/latest/)パッケージをインストールします。

```bash
$ sudo pip install requests
```

`git clone`したディレクトリに移動して、[bme280_data.py](https://github.com/IDCFChannel/bme280-meshblu-py/blob/master/bme280_data.py)を使います。

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
    }

    payload = {"trigger":"on"}

    while True:
        sleep(5)
        retval = sensing()
        if retval:
             print("temperature: {}".format(retval["temperature"]))
             if float(retval["temperature"]) > conf["THRESHOLD"]:
                 print("threshold over: {0} > {1}".format(float(retval["temperature"]),
                                     conf["THRESHOLD"]))
                 r = requests.post(url, headers=headers, data=payload)

if __name__ == '__main__':
    main()
```

センサーデータの閾値を監視するコードは以下です。config.pyに設定した`THRESHOLD`の値とセンサーから取得した気温(temperature)を比較します。`THRESHOLD`は`27.0`に設定しているので気温が27.0より高い場合にトリガーの発火条件となります。

```python
    while True:
        retval = sensing()
        if retval:
             print("temperature: {}".format(retval["temperature"]))
             if float(retval["temperature"]) > conf["THRESHOLD"]:
                 print("threshold over: {0} > {1}".format(float(retval["temperature"]),
                                     conf["THRESHOLD"]))
                 r = requests.post(url, headers=headers, data=payload)
        sleep(5)
```

トリガーの発火条件を満たしたことをmyThingsサーバーに伝えるため、IDCFチャンネルサーバーにHTTP POSTします。IDCFチャンネルサーバーのIPアドレスに続けて`/data/{trigger-1のuuid`を指定します。

```python
    url = "http://{0}/data/{1}".format(conf["IDCF_CHANNEL_URL"],
                                       conf["TRIGGER_1_UUID"])
```

完成したURLに認証用のHTTPヘッダを追加してHTTP POSTします。POSTするデータは現在IDCFチャンネルのトリガーからアクションに値として渡すことができないので、なくても構いません。

```bash
http://210.140.162.58/data/21c83792-b25e-4ae7-a627-714af57a1a4b
```

### トリガーの発火とアクションの実行

config.pyの`THRESHOLD`の値は閾値監視を通るように値を調整してからプログラムを実行します。

```bash
$ sudo ./bme280_data.py
temperature: 27.26
threshold over: 27.26 > 27.0
```

閾値を超えたらCtrl-Cを押してプログラムを終了させます。

myThingsアプリの組合せを開いて「手動実行」ボタンを押します。トリガーの発火条件を満たしているのでGmailアクションが実行され、メールが届きます。

![gmail-received.png](/2015/09/16/mythings-idcfchannel-trigger-edge/gmail-received.png)

## /dataにPOSTするとMongoDBにレコードが作成される

トリガー発火の条件を満たして`/data/{trigger-1のuuid}`にHTTP POSTすると、IDCFチャンネルサーバーのMongoDBにレコードが作成されます。

IDCFチャンネルサーバーが実行されているディレクトリに移動してから、MongoDBのコンテナにアクセスします。

```bash
$ cd ~/iot_apps/meshblu-compose
$ docker exec -it meshblucompose_mongo_1 mongo skynet
>
```

`data`コレクションの最新の値を確認します。trigger-1のuuidを持ち、先ほどトリガー発火条件を満たした時刻にHTTP POSTしたレコードが登録されています。

```bash
> db.data.find().sort({ $natural: -1 }).limit(1)
{ "_id" : ObjectId("55fa682d06986a0e000058da"), "trigger" : "on", "ipAddress" : "210.xxx.xxx.xxx", "uuid" : "21c83792-b25e-4ae7-a627-714af57a1a4b", "timestamp" : "2015-09-17T07:13:49.438Z" }
```

myThingsアプリの組合せで指定したトリガーのレコードが作成されているか、myThingsサーバーは15分間隔で`data`コレクションを確認しています。myThingsアプリの「手動実行」ボタンを押した場合はすぐに確認します。

前回確認した時刻より新しいレコードが見つかると、トリガー発火条件を満たしたと判断してmyThingsサーバーは設定されたアクションを実行します。
