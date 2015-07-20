title: "MeshbluでオープンソースのIoTをはじめよう - Part2: BME280の環境データをfreeboardに表示する"
date: 2015-07-07 07:26:13
categories:
 - IoT
tags:
 - IoT
 - freeboard
 - BME280
 - Meshblu
 - RaspberryPi
 - IDCFクラウド
description: 今回はRaspberry Pi 2に配線した環境センサのBME280からMeshbluのブローカーにMQTTを使ってデータを送信してみます。。取得した温度、湿度、気圧のデータは、オープンソースのIoT用ダッシュボードのfreeboadに表示します。簡単にブラウザからリアルタイムのグラフを作成することができます。
---

今回はRaspberry Pi 2に配線した環境センサの[BME280](https://www.switch-science.com/catalog/2236/)からMeshbluのブローカーにMQTTを使ってデータを送信してみます。。取得した温度、湿度、気圧のデータは、オープンソースのIoT用ダッシュボードの[freeboad](https://github.com/Freeboard/freeboard)に表示します。簡単にブラウザからリアルタイムのグラフを作成することができます。


<!-- more -->

## Meshblu

[前回](/2015/07/06/meshblu-compose-start/)の手順でMeshbluサーバーの構築とuuidを登録済にしておきます。今回は登録してある10個のuuidから以下の3つを割り当てて使います。

* `trigger-1` : Raspberry Pi 2からBME280の環境データを送信するuuid
* `action-1`  : Dockerホスト上で、MQTT subscribeするuuid
* `action-2`  : freeboard上で、WebSocket subscribeするuuid

それぞれのtokenとuuidを`list`コマンドから確認します。

```bash
$ cd ~/iot_apps/meshblu-compose
$ docker-compose run --rm iotutil list

> iotutil@0.0.1 start /app
> node app.js "list"

┌───────────┬──────────┬──────────────────────────────────────┐
│ keyword   │ token    │ uuid                                 │
├───────────┼──────────┼──────────────────────────────────────┤
│ trigger-1 │ 89ad09fd │ 9ca04922-88a3-404e-8709-f6a544763c35 │
├───────────┼──────────┼──────────────────────────────────────┤
...
├───────────┼──────────┼──────────────────────────────────────┤
│ action-1  │ ad8a3c60 │ a1e99a5f-acb6-4210-87c4-d195ea3d9f69 │
├───────────┼──────────┼──────────────────────────────────────┤
│ action-2  │ 35e93e3e │ 8eb3a4c0-5fb0-456c-9b90-44e59c40f5ad │
├───────────┼──────────┼──────────────────────────────────────┤
...
```

`trigger-1`から`action-2`へメッセージが送信できるように設定します。

```bash
$ docker-compose run --rm iotutil whiten -- -f trigger-1 -t action-2

> iotutil@0.0.1 start /app
> node app.js "whiten" "-f" "trigger-1" "-t" "action-2"

trigger-1 can send message to action-2
```

Dockerホストに別のシェルを開きmosquott_subコマンド使います。`action-2`のuuidをトピック名にしていして、`action-2`のuuidへのメッセージをsubscribeします。

```bash
$ mosquitto_sub \
    -h localhost \
    -p 1883  \
    -t a1e99a5f-acb6-4210-87c4-d195ea3d9f69  \
    -u a1e99a5f-acb6-4210-87c4-d195ea3d9f69  \
    -P ad8a3c60  \
    -d
```

## Raspberry Pi

BME280の環境センサーは[前回](/2015/06/30/raspberrypi-bme280/)の手順に従ってRaspberry Pi 2に配線します。

![bme280-plus.png](/2015/07/07/meshblu-compose-bme280-freeboard/bme280-plus.png)

`i2cdetect`コマンドを使いセンサーが正しく配線されていることを確認します。

```bash
$ sudo i2cdetect -y 1
     0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f
00:          -- -- -- -- -- -- -- -- -- -- -- -- --
10: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
20: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
30: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
40: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
50: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
60: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
70: -- -- -- -- -- -- 76 --
```

こちらの[リポジトリ](https://github.com/IDCFChannel/bme280-meshblu-py)からPythonのサンプルコードをダウンロードします。

```bash
$ mkdir -p ~/python_apps && cd ~/python_apps
$ git clone https://github.com/IDCFChannel/bme280-meshblu-py.git
$ cd bme280-meshblu-py.git
```

config.py.defaultの名前をconfig.pyに変更して編集します。

```bash
$ mv config.py.default config.py
```

uuidとtoken情報を入力します。`IDCF_CHANNEL_URL`のアドレスは環境にあわせて設定します。

* TRIGGER_UUID (`trigger-1`のuuid)    : 9ca04922-88a3-404e-8709-f6a544763c35
* TRIGGER_TOKEN (`trigger-1`のtoken)  : 89ad09fd
* ACTION_UUID (`action-1`のuuid)      : a1e99a5f-acb6-4210-87c4-d195ea3d9f69
* FREEBOARD_UUID (`action-2`のuuid)   : 8eb3a4c0-5fb0-456c-9b90-44e59c40f5ad


```py ~/python_apps/bme280-meshblu-py/config.py
conf = {
     "IDCF_CHANNEL_URL": "210-140-167-168.jp-east.compute.idcfcloud.com",
     "TRIGGER_UUID": "9ca04922-88a3-404e-8709-f6a544763c35",
     "TRIGGER_TOKEN": "89ad09fd",
     "ACTION_UUID": "a1e99a5f-acb6-4210-87c4-d195ea3d9f69",
     "FREEBOARD_UUID": "8eb3a4c0-5fb0-456c-9b90-44e59c40f5ad"
}
```

bme280_publish.pyをroot権限で実行します。JSON形式のメッセージに設定する`devices`ディレクティブにはACTION_UUID (`action-1`)とFREEBOARD_UUID(`action-2`)のuuidを指定しています。Raspberry Pi 2からはこの2つのuuidを宛先にメッセージを送信します。

```py ~/python_apps/bme280-meshblu-py/bme280_publish.py
#!/usr/bin/python
# -*- coding: utf-8 -*-
import paho.mqtt.client as mqtt
from time import sleep
import json
import sys
import bme280
from config import conf

def sensing():
    return bme280.readData()

def on_connect(client, userdata, rc):
    print("Connected with result code {}".format(rc))

def on_publish(client, userdata, mid):
    print("publish: {}".format(mid))

def main():
    client = mqtt.Client(client_id='',
                         clean_session=True, protocol=mqtt.MQTTv311)

    client.username_pw_set(conf["TRIGGER_UUID"], conf["TRIGGER_TOKEN"])

    client.on_connect = on_connect
    client.on_publish = on_publish

    client.connect(conf["IDCF_CHANNEL_URL"], 1883, 60)

    while True:
        retval = sensing()
        if retval:
             message = json.dumps({"devices":
                              [conf["ACTION_UUID"],
                               conf["FREEBOARD_UUID"]],
                              "payload": retval})
             print(message)
             client.publish("message",message)
        sleep(5)

if __name__ == '__main__':
    main()
```

bme280_publish.pyプログラムをroot権限で実行します。5秒間隔で指定した2つのuuidにメッセージをpublishします。

```bash
$ sudo python bme280_publish.py
{"payload": {"pressure": "1009.91", "temperature": "27.85", "humidity": "60.16"}, "devices": ["a1e99a5f-acb6-4210-87c4-d195ea3d9f69", "8eb3a4c0-5fb0-456c-9b90-44e59c40f5ad"]}
publish: 1
{"payload": {"pressure": "1009.90", "temperature": "27.87", "humidity": "59.90"}, "devices": ["a1e99a5f-acb6-4210-87c4-d195ea3d9f69", "8eb3a4c0-5fb0-456c-9b90-44e59c40f5ad"]}
publish: 2
...
```

### Dockerホストで確認

Dockerホストのmosquitto_subが起動しているシェルに戻ります。Raspberry Pi 2から環境データのメッセージをsubscribeできました。

```bash
...
Client mosqsub/57336-minion1.c received PUBLISH (d0, q0, r0, m0, 'a1e99a5f-acb6-4210-87c4-d195ea3d9f69', ... (72 bytes))
{"data":{"pressure":"1009.91","temperature":"27.78","humidity":"59.11"}}
Client mosqsub/57336-minion1.c received PUBLISH (d0, q0, r0, m0, 'a1e99a5f-acb6-4210-87c4-d195ea3d9f69', ... (72 bytes))
{"data":{"pressure":"1009.91","temperature":"27.78","humidity":"59.11"}}
```

## freeboard

Dockerホストでもう一つ別のシェルを開きます。Doker Composeの[設定ファイル](https://github.com/IDCFChannel/meshblu-compose/blob/master/docker-compose.yml)には`freeboard`のサービスを定義しています。

```yml ~/iot_apps/meshblu-compose/docker-compose.yml
freeboard:
  build: ./freeboard
  volumes:
    - /etc/localtime:/etc/localtime:ro
  ports:
    - 8088:8080
```

`freeboard`サービスをupします。

```bash
$ docker-compose up -d freeboard
```

### 自己署名のSSL通信について

OpenRestyのDockerコンテナを起動するときに、SSL通信用の証明書を自己署名で作成しています。ダッシュボードを表示するブラウザがWebSocketを使ってセンサーデータをsubscribeするときもSSL通信を行います。ブラウザで予めMeshbluのURLに対してhttps通信を行い証明書の利用を許可してください。

例: https://210-140-167-168.jp-east.compute.idcfcloud.com


![freeboard-ssl.png](/2015/07/07/meshblu-compose-bme280-freeboard/freeboard-ssl.png)


### データソースの編集

[リポジトリ](https://github.com/IDCFChannel/bme280-meshblu-py/blob/master/freeboard-bme280.json)からJSON形式の設定ファイルをダウンロードしておきます。docker-compose.ymlに設定したポート番号をブラウザで開き、さきほどの設定ファイルをアップロードします。

![freeboard-init.png](/2015/07/07/meshblu-compose-bme280-freeboard/freeboard-init.png)


ダッシュボード設定の雛形がアップロードされました。`bme280`をクリックしてデータソースの設定を行います。

![freeboard-datasource-bme280.png](/2015/07/07/meshblu-compose-bme280-freeboard/freeboard-datasource-bme280.png)

ダッシュボードの更新ボタンを押します。入力が終わったらSAVEボタンを押して設定画面を閉じます。

![freeboard-datasource-bme280-uuid-password.png](/2015/07/07/meshblu-compose-bme280-freeboard/freeboard-datasource-bme280-uuid-password.png)


Raspberry Pi 2から送信された環境データがダッシュボードに表示されました。

![freeboard-reload.png](/2015/07/07/meshblu-compose-bme280-freeboard/freeboard-reload.png)





