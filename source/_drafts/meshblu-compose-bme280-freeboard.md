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
---

ダッシュボードのfreeboadに表示します。


<!-- more -->

## Meshblu

最初にMesubluも[前回](/2015/07/06/meshblu-compose-start/)の手順でサーバーを起動しておきます。今回は以下の3つのデバイスを使います。

* `trigger-1`: Raspberry Pi 2からBME280の環境データを送信するデバイス 
* `action-1`: Dockerホスト上で、MQTT subscribeするデバイス
* `action-2`: freeboard上で、WebSocket subscribeするデバイス

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
Dockerホスト上でmosquott_subコマンド使い、MQTTのsubscribeをしておきます。

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

最初に[前回](/2015/06/30/raspberrypi-bme280/)の手順でBME280の環境センサーをRaspberry Pi 2に配線します。

[!bme280-plus.png](/2015/07/07/meshblu-compose-bme280-freeboard/bme280-plus.png)

`i2cdetect`コマンドを使い正常に接続していることを確認します。

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

リポジトリからサンプルコードをダウンロードします。

```
$ mkdir -p ~/python_apps && cd ~/python_apps
$ git clone https://github.com/IDCFChannel/bme280-meshblu-py.git
$ cd bme280-meshblu-py.git
```

config.pyを編集します。それぞれ上記で確認したデバイスの情報を入力します。`IDCF_CHANNEL_URL`のアドレスは環境に併せて設定します。


```py ~/python_apps/bme280-meshblu-py/config.py
conf = {
     "IDCF_CHANNEL_URL": "210-140-167-168.jp-east.compute.idcfcloud.com",
     "TRIGGER_UUID": "9ca04922-88a3-404e-8709-f6a544763c35",
     "TRIGGER_TOKEN": "89ad09fd",
     "ACTION_UUID": "a1e99a5f-acb6-4210-87c4-d195ea3d9f69",
     "FREEBOARD_UUID": "8eb3a4c0-5fb0-456c-9b90-44e59c40f5ad"
}
```

bme280-meshblu-pyを実行します。messageの`devices`ディレクティブにはACTION_UUID(action-1)とFREEBOARD_UUID(action-2)のuuidを指定しているので、この2つのデバイスにメッセージを送信します。

```py ~/python_apps/bme280-meshblu-py/bme280-meshblu-py
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

root権限でプログラムを実行します。5秒間隔で指定した2つのデバイスにメッセージをpublishします。

```bash
$ sudo python bme280_publish.py
{"payload": {"pressure": "1009.91", "temperature": "27.85", "humidity": "60.16"}, "devices": ["a1e99a5f-acb6-4210-87c4-d195ea3d9f69", "8eb3a4c0-5fb0-456c-9b90-44e59c40f5ad"]}
publish: 1
{"payload": {"pressure": "1009.90", "temperature": "27.87", "humidity": "59.90"}, "devices": ["a1e99a5f-acb6-4210-87c4-d195ea3d9f69", "8eb3a4c0-5fb0-456c-9b90-44e59c40f5ad"]}
publish: 2
...
```

Dockerホストのmosquitto_subが起動しているシェルを確認します。メッセージがsubscribeできました。

```bash
Client mosqsub/57336-minion1.c received PUBLISH (d0, q0, r0, m0, 'a1e99a5f-acb6-4210-87c4-d195ea3d9f69', ... (72 bytes))
{"data":{"pressure":"1009.91","temperature":"27.78","humidity":"59.11"}}
Client mosqsub/57336-minion1.c received PUBLISH (d0, q0, r0, m0, 'a1e99a5f-acb6-4210-87c4-d195ea3d9f69', ... (72 bytes))
{"data":{"pressure":"1009.91","temperature":"27.78","humidity":"59.11"}}
```

## freeboard

Dockerホストで別のシェルを開きます。docker-compose.ymlには`freeboard`のサービスを定義してあるので簡単に起動することができます。ポート番号は環境に合わせて変更します。

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

SSL通信の証明書は自己署名を使っているのでブラウザを開いて最初に承認しておきます。

https://210.140.167.168

docker-compose.ymlに定義したfreeboardのポートをブラウザで開きます。

http://210.140.167.168:8088/

データソースに`Octoblu`を選択して`action-2`のデバイス情報を入力します。
