title: "myThingsをはじめよう - Part8: トリガーの閾値監視をIDCFクラウド側で行う"
date: 2015-09-17 14:40:21
categories:
 - IoT
tags:
 - IoT
 - Meshblu
 - myThings
 - BME280
 - IDCFクラウド
description: Raspberry Piのプログラムからセンサーデータをmessageトピックにpublishして、IDCFクラウドのサーバー側のプログラムでsubscribeと閾値監視を行うパターンです。トリガーの発火条件をRaspberry Piのセンサーデータに加えてサーバー側で他のデータと組み合わせたい場合に使います。一度データをsubscribeしているので、閾値監視以外にもデータベースに保存したり、さらに別のデバイスにメッセージをpublishすることもできます。
---

Raspberry Piのプログラムからセンサーデータをmessageトピックにpublishして、IDCFクラウドのサーバー側のプログラムでsubscribeと閾値監視を行うパターンです。トリガーの発火条件をRaspberry Piのセンサーデータに加えてサーバー側で他のデータと組み合わせたい場合に使います。一度データをsubscribeしているので、閾値監視以外にもデータベースに保存したり、さらに別のデバイスにメッセージをpublishすることもできます。

<!-- more -->

## はじめに

### 作業の流れ

以下の手順で作業を進めていきます。

1. Raspberry PiでBME280センサーデータを取得する
1. Raspberry PiからIDCFチャンネルサーバーの`message`トピックにMQTT publishする
1. IDCFクラウドのサーバープログラムで`actionのuuid`トピックをMQTT subscribeする
1. IDCFクラウドのサーバープログラムで閾値監視をする
1. IDCFクラウドのサーバープログラムからHTTP POST `/data/{triggerのuuid}`する
1. myThingsアプリから組合せを「手動実行」する
1. Gmailにメールが届く


## trigger-1とaction-1の確認

IDCFチャンネルサーバーの仮想マシンにログインして`list`コマンドを実行します。今回使用するtrigger-1とaction-1のuuidとtokenを確認します。


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
│ action-1  │ 8a781e76 │ 3a78814a-6879-4543-bacf-9a206cd951a6 │
...
```

## Raspberry Pi側のプログラム

GitHubの[リポジトリ](https://github.com/IDCFChannel/bme280-meshblu-py)から最新のコードをダウンロードします。

```bash
$ cd ~/python_apps/bme280-meshblu-py
$ git pull
```

新規の場合はgit cloneします。

```bash
$ git clone https://github.com/IDCFChannel/bme280-meshblu-py.git
```

プログラムの実行に必要なパッケージをrequirements.txtからインストールします。

```bash
$ cat requirements.txt
paho-mqtt==1.1
requests==2.7.0
$ sudo pip install -r requirements.txt
```

root権限でプログラムを実行します。5秒間隔で計測したセンサーデータがpublishされます。

```python bme280_pubcloud.py
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
    # MQTT
    client = mqtt.Client(client_id='',
	                 clean_session=True, protocol=mqtt.MQTTv311)
    client.username_pw_set(conf["TRIGGER_1_UUID"], conf["TRIGGER_1_TOKEN"])
    client.on_connect = on_connect
    client.on_publish = on_publish
    client.connect(conf["IDCF_CHANNEL_URL"], 1883, 60)

    while True:
        sleep(5)
        retval = sensing()
        if retval:
             message = json.dumps({"devices": conf["ACTION_1_UUID"],
                                   "payload": retval})
             print(message)
             client.publish("message", message)
if __name__ == '__main__':
    main()
```

config.pyは環境にあわせて設定します。trigger-1とaction-1を使います。

```python config.py
conf = {
     "IDCF_CHANNEL_URL": "210.140.162.58",
     "TRIGGER_1_UUID": "21c83792-b25e-4ae7-a627-714af57a1a4b",
     "TRIGGER_1_TOKEN": "d74ebedf",
     "ACTION_1_UUID": "3a78814a-6879-4543-bacf-9a206cd951a6",
     "THRESHOLD": 27.0
}
```

### trigger-1からaction-1にpublishする

現状のMeshbluのMQTTブローカーはpublishとsubscribeに同じuuidを使うと2回メッセージが届くようです。そのためこのサンプルではpublishとsubscribeで別のuuidを選択します。publishはtrigger-1、subscribeはaction-1のuuidを使います。

* publishするuuid: trigger-1
* subscribeするuuid: action-1

前回と同様にBME280からセンサーデータを計測し、5秒間隔でJSON形式のデータをpublishします。

### インストールと実行

GitHubの[リポジトリ](https://github.com/IDCFChannel/bme280-meshblu-py)から最新のコードをダウンロードします。

```bash
$ cd ~/python_apps/bme280-meshblu-py
$ git pull
```

新規の場合はgit cloneします。

```bash
$ git clone https://github.com/IDCFChannel/bme280-meshblu-py.git
```

プログラムの実行に必要なパッケージをrequirements.txtからインストールします。

```bash
$ cat requirements.txt
paho-mqtt==1.1
requests==2.7.0
$ sudo pip install -r requirements.txt
```

root権限でプログラムを実行します。5秒間隔で計測したセンサーデータがpublishされます。

```bash
$ cat requirements.txt
paho-mqtt==1.1
requests==2.7.0
$ sudo pip install -r requirements.txt
```

root権限でプログラムを実行します。5秒間隔で

```bash
$ sudo python bme280_pubcloud.py
{"payload": {"pressure": "1008.85", "temperature": "30.24", "humidity": "39.20"}, "devices": "3a78814a-6879-4543-bacf-9a206cd951a6"}
publish: 1
{"payload": {"pressure": "1008.87", "temperature": "30.25", "humidity": "39.20"}, "devices": "3a78814a-6879-4543-bacf-9a206cd951a6"}
publish: 2
...
```

## IDCFクラウドのサーバープログラム

[bme280_subcloud.py](https://github.com/IDCFChannel/meshblu-compose/blob/master/bme280/bme280_subcloud.py)がMQTTのメッセージをsubscribeするプログラムです。


```python bme280_subcloud.py
# -*- coding: utf-8 -*-
import paho.mqtt.client as mqtt
import json
import requests
from config import conf

url = "http://{0}/data/{1}".format(conf["IDCF_CHANNEL_URL"],
                                       conf["TRIGGER_1_UUID"])
headers = {
    "meshblu_auth_uuid": conf["TRIGGER_1_UUID"],
    "meshblu_auth_token": conf["TRIGGER_1_TOKEN"]
}

payload = "trigger on"

def on_connect(client, userdata, rc):
    print("Connected with result code {}".format(rc))
    client.subscribe(conf["ACTION_1_UUID"])

def on_subscribe(client, userdata, mid, granted_qos):
    print("Subscribed mid: {0},  qos: {1}".format(str(mid), str(granted_qos)))

def on_message(client, userdata, msg):
    payload = json.loads(msg.payload)
    data = payload["data"]
    if data:
        retval = data["payload"]
        if retval:
            print("-"*10)
            print("temperature: {}".format(retval["temperature"]))
            if float(retval["temperature"]) > conf["THRESHOLD"]:
                print("threshold over: {0} > {1}".format(float(retval["temperature"]),
                                                         conf["THRESHOLD"]))
                r = requests.post(url, headers=headers, data=payload)

def main():
    # MQTT
    client = mqtt.Client(client_id='',
                         clean_session=True, protocol=mqtt.MQTTv311)
    client.username_pw_set(conf["ACTION_1_UUID"], conf["ACTION_1_TOKEN"])
    client.on_connect = on_connect
    client.on_subscribe = on_subscribe
    client.on_message = on_message

    client.connect(conf["IDCF_CHANNEL_URL"], 1883, 60)
    client.loop_forever()

if __name__ == '__main__':
    main()
```

### action-1でsubscribeする

Raspberry Pi側のプログラムではtrigger-1からaction-1にMQTTのメッセージをpublishしました。クラウド側のプログラムでもMQTTのメッセージはaction-1でsubscribeします。


```python
def on_connect(client, userdata, rc):
    print("Connected with result code {}".format(rc))
    client.subscribe(conf["ACTION_1_UUID"])
```

### trigger-1にPOSTしてmyThingsのトリガーを発火する

少々ややこしくなりますがsubscribeしたセンサーデータが閾値を超えた場合、今度はtrigger-1のuuidを`/data/{trigger-1のuuid}`に入れたURLに任意のデータをPOSTします。

```python
url = "http://{0}/data/{1}".format(conf["IDCF_CHANNEL_URL"],
                                       conf["TRIGGER_1_UUID"])
```

現在のmyThingsはIDCFチャンネルをトリガーにしてアクションにデータを渡すことができないため、POSTするデータは任意です。POSTしてmyThings連携用のデータベースにレコードを作成することが目的です。

```python
payload = "trigger on"
```

### meshblu-composeプログラムの更新

GitHubの[リポジトリ](https://github.com/IDCFChannel/meshblu-compose)から最新のコードを取得します。現在起動しているコンテナは破棄して新しいコンテナを起動します。`mongo`と`redis`ディレクトリに永続化データが保存されているので、このディレクトリを削除しなければ以前のデータを引き続き使えます

```bash
$ cd ~/iot_apps/meshblu-compose/
$ docker-compose stop
$ docker-compose rm
$ git pull
```

新しいプログラムをダウンロードしたらmeshblu-composeを起動します。

```bash
$ docker-compose up -d openresty
```

### イメージのビルドと実行

メッセージのsubscribeと閾値監視を行うIDCFクラウド側のプログラムをビルドします。[docker-compose.yml](https://github.com/IDCFChannel/meshblu-compose/blob/master/docker-compose.yml)にサービスが定義してあります。リポジトリは[こちら](https://github.com/IDCFChannel/meshblu-compose/tree/master/bme280)です。

```yml docker-compose.yml
bme280:
  build: ./bme280
  volumes:
    - /etc/localtime:/etc/localtime:ro
```

オフィシャル[Python](https://hub.docker.com/_/python/)のベースイメージを使います。ONBUILDのベースイメージはビルドするときにカレントディレクトリのrequirements.txtをCOPYして必要なパッケージをインストールしてくれます。

```bash Dockerfile
FROM python:2.7

RUN mkdir -p /usr/src/app
WORKDIR /usr/src/app

ONBUILD COPY requirements.txt /usr/src/app/
ONBUILD RUN pip install --no-cache-dir -r requirements.txt

ONBUILD COPY . /usr/src/app
```

またONBUILDでカレントディレクトリのファイルがCOPYされるので、[Dockerfile](https://github.com/IDCFChannel/meshblu-compose/blob/master/bme280/Dockerfile)では実行したいプログラムをCMDに指定します。

```bash Dockerfile
FROM python:2-onbuild
CMD [ "python", "./bme280_subcloud.py" ]
```

Dockerイメージをビルドします。

```bash
$ docker-compose build bme280
```

Raspberry PiでBME280から計測したデータをmessageトピックにMQTT publishしてIDCFクラウド側でMQTT subscribeします。センサーデータが閾値を超えた場合に`/data/{triggerのuuid}`へHTTP POSTするとmyThingsのトリガーが発火されます。

```bash
$ docker-compose run --rm bme280
Connected with result code 0
Subscribed mid: 1,  qos: (0,)
----------
temperature: 30.07
threshold over: 30.07 > 27.0
----------
temperature: 30.07
threshold over: 30.07 > 27.0
```
