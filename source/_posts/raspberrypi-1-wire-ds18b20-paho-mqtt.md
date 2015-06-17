title: "Raspberry PiのPythonでDS18B20のセンシングデータをPahoのMQTTからpublishする"
date: 2015-04-14 23:00:01
categories:
 - IoT
tags:
 - RaspberryPi
 - MQTT
 - Paho
 - DS18B20
 - Python
 - Meshblu
description: 前回Raspberry Pi上で1-Wireデジタル温度センサのDS18B20からセンシングデータをPythonを使い取得するサンプルを作りました。次に取得したデータをPahoのPythonクライアントを使ってMQTTブローカーであるMeshbluにpublishしてみます。
---

[前回](/2015/04/12/raspberrypi-1-wire-ds18b20-python/)Raspberry Pi上で1-Wireデジタル温度センサのDS18B20からセンシングデータをPythonを使い取得するサンプルを作りました。次に取得したデータを[Paho](http://www.eclipse.org/paho/)のPythonクライアントを使ってMQTTブローカーであるMeshbluにpublishしてみます。

<!-- more -->

## Paho

パッケージのインストールにpipを使います。

``` bash
$ curl https://bootstrap.pypa.io/get-pip.py -o - | sudo python
```

PahoのPythonクライアントである[paho-mqtt](https://pypi.python.org/pypi/paho-mqtt)をインストールします。

```　bash
$ sudo pip install paho-mqtt
...
Successfully installed paho-mqtt-1.1
```

## Meshbluのデバイス登録

最初に今回のサンプルで使うDS18B20のデバイスをMeshbluに2つ登録します。センシングデータ保存用のデバイスIDと閾値監視用のデバイスIDを分けています。最初にtoken用にランダム文字列を作成します。

``` bash
$ python
Python 2.7.6 (default, Sep  9 2014, 15:04:36)
[GCC 4.2.1 Compatible Apple LLVM 6.0 (clang-600.0.39)] on darwin
Type "help", "copyright", "credits" or "license" for more information.
>>> import random
>>> num = 8
>>> arr = list('0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ')
>>> print  "".join([random.choice(arr) for i in range(num)])
DrwWzKBQ
>>> print  "".join([random.choice(arr) for i in range(num)])
SiwPDhUi
```


### センシングデータ用のデバイス - sensing-dev

5秒間隔でDS18B20から計測した温度をpublishします。あとでInfluxDBにブリッジしたり、WebSoketでブラウザにリアルタイムでグラフを描画したりする予定です。Meshbluにデバイス登録して所有権を取得します。

``` bash
$ curl -X POST \
  http://localhost/devices \
  -d "name=sensing-dev&uuid=sensing-dev&token=DrwWzKBQ"
$ curl -X PUT \
  http://localhost/claimdevice/sensing-dev \
  --header "meshblu_auth_uuid: sensing-dev" \
  --header "meshblu_auth_token: DrwWzKBQ"
```

### 閾値監視用のデバイス - threshold-dev

Rasbperry Pi上で閾値監視をします。指定した温度以上場合にLチカする想定でMongoDBに永続化します。上記と同じようにデバイスを登録します。

``` bash
$ curl -X POST \
  http://localhost/devices \
  -d "name=threshold-dev&uuid=threshold-dev&token=SiwPDhUi"
$ curl -X PUT \
  http://localhost/claimdevice/threshold-dev \
  --header "meshblu_auth_uuid: threshold-dev" \
  --header "meshblu_auth_token: SiwPDhUi"
```

## サンプル

5秒間隔でDS18B20モジュールから温度を計測して摂氏25度以上の場合MQTTでMesubluのブローカーにpublishするサンプルです。Raspberry Pi上で実行するPythonプログラムを用意します。

### プログラム

PahoのMQTTクライアントのusernameは`threshold-dev`になります。


```python ~/python_apps/w1therm-mqtt-publish.py
# -*- coding: utf-8 -*-
import paho.mqtt.client as mqtt
from time import sleep
from w1thermsensor import W1ThermSensor
import json

def sensing():
    sensor = W1ThermSensor()
    return sensor.get_temperature()

def on_publish(client, userdata, mid):
    print("publish: {0}".format(mid))

def main():
    client = mqtt.Client(client_id='',clean_session=True, protocol=mqtt.MQTTv311)
    client.username_pw_set("threshold-dev","SiwPDhUi")
    client.connect("MESUBLU_BROKER", 1883, 60)
    client.on_publish = on_publish

    while True:
        sensing_value = sensing()
        sensing_message = json.dumps({"devices":"sensing-dev",
                                      "payload":{"temperature":sensing_value}
                                    })
        client.publish("message",sensing_message)

        if sensing_value > 25.0:
            threshold_message = json.dumps({"red":"on"})
            client.publish("data",threshold_message)
        sleep(5)

if __name__ == '__main__':
    main()
```

### 実行

Mesubluのローカル上でMosquittoを使い`sensing-dev` topicをsubscribeします。

``` bash
$ mosquitto_sub \
  -h localhost \
  -p 1883 \
  -t sensing-dev \
  -u sensing-dev \
  -P DrwWzKBQ \
  -d
Client mosqsub/15463-minion1.c sending CONNECT
Client mosqsub/15463-minion1.c received CONNACK
Client mosqsub/15463-minion1.c sending SUBSCRIBE (Mid: 1, Topic: sensing-dev, QoS: 0)
Client mosqsub/15463-minion1.c received SUBACK
Subscribed (mid: 1): 0
```

少しわかりづらいですがMesubluではメッセージをpublishする場合、topic名は`message`が固定になります。メッセージ本文の`devices`キーに指定したtopic名でsubscribeします。

```
{"devices":"sensing-dev",
  "payload":{"temperature":sensing_value}
}
```                            

Raspberry PiでPythonプログラムを実行します。5秒間隔で`message` topicと`data` topicにpublishされます。

``` bash
$ python w1therm-mqtt-threshold.py
publish: 1
publish: 2
publish: 3
publish: 4
publish: 5
publish: 6
```

Meshbluローカルのmosquitto_subを確認するとメッセージが届きました。

``` bash
...
Client mosqsub/15463-minion1.c received PUBLISH (d0, q0, r0, m0, 'sensing-dev', ... (112 bytes))
{"topic":"message","data":{"payload":{"temperature":26.062},"devices":"sensing-dev","fromUuid":"threshold-dev"}}
Client mosqsub/15463-minion1.c received PUBLISH (d0, q0, r0, m0, 'sensing-dev', ... (112 bytes))
{"topic":"message","data":{"payload":{"temperature":26.062},"devices":"sensing-dev","fromUuid":"threshold-dev"}}
Client mosqsub/15463-minion1.c received PUBLISH (d0, q0, r0, m0, 'sensing-dev', ... (112 bytes))
```

`data` topicにpublishしたデータはMongoDBに永続化されます。`data` topicはREST APIでクエリできるので非同期メッセージの場合に便利に使えます。

``` bash
$ mongo skynet
> db.data.find().sort({ $natural: -1 }).limit(5)
{ "timestamp" : "2015-04-14T03:00:53.050Z", "red" : "on", "fromUuid" : "threshold-dev", "uuid" : "threshold-dev", "_id" : ObjectId("55386065f8b6800700f67987") }
{ "timestamp" : "2015-04-14T03:00:47.184Z", "red" : "on", "fromUuid" : "threshold-dev", "uuid" : "threshold-dev", "_id" : ObjectId("5538605ff8b6800700f67986") }
...
```
