title: "MeshbluでオープンソースのIoTをはじめよう - Part3: BeagleBone BlackからDS18B20の温度データをfreeboardに表示する"
date: 2015-07-12 23:11:30
tags:
 - Meshblu
 - DS18B20
 - Supervisor
 - freeboard
 - IDCFクラウド
 - WebSocket
 - MQTT
description: BME280]の環境センサを使ってRaspberry Pi 2から環境データをMQTT publishしました。同様にBeagleBone BlackとDS18B20の組み合わせでも試してみます。MeshbluがMQTTとWebSocketをブリッジします。ブラウザ上のfreeboadがJavaScriptでWebSocketをsubscribeします。

---

[BME280](https://www.switch-science.com/catalog/2236/)の環境センサを使って[Raspberry Pi 2から環境データをMQTT publish](/2015/07/07/meshblu-compose-bme280-freeboard/)しました。同様にBeagleBone BlackとDS18B20の組み合わせでも試してみます。MeshbluがMQTTとWebSocketをブリッジします。ブラウザ上の[freeboad](https://github.com/Freeboard/freeboard)がJavaScriptでWebSocketをsubscribeします。

<!-- more -->

## DS18B20とBeagleBone Black

BeagleBone Blackで1-wireのDS18B20を配線するのはかなり面倒です。センサーデータを取得するなら同じARM LinuxでもRaspberry Piの方が簡単です。ただBeagleBone Blackには[BoneScript](https://github.com/jadonk/bonescript)や[OctalBoneScrit](https://github.com/theoctal/octalbonescript)といったRaspberry Piにはない遊び方ができるのでとても気に入っています。

## Meshbluのセットアップ

今回作成したリポジトリは[こちら](https://github.com/IDCFChannel/glow-light-py)です。MesubluのブローカーはDocker Composeで構成管理した[Mesublu Compose](https://github.com/IDCFChannel/meshblu-compose)を用意しました。予めDockerホストマシンにセットアップしておきます。

```bash
$ cd ~/iot_util
$ git clone --recursive https://github.com/IDCFChannel/meshblu-compose
$ cd meshblu-compose
$ ./bootstrap.sh
```
インストール後にregisterを実行してBeagleBone Blackとの紐付けに必要なuuid作成します。

```bash
$ docker-compose run --rm iotutil register
```

`list`コマンドを実行すると登録されたuuidとtokenがリストで表示されます。

```bash
$ docker-compose run --rm iotutil list

> iotutil@0.0.1 start /app
> node app.js "list"

┌───────────┬──────────┬──────────────────────────────────────┐
│ keyword   │ token    │ uuid                                 │
├───────────┼──────────┼──────────────────────────────────────┤
│ trigger-1 │ 49f74df6 │ c1871f89-d82e-4d49-ac43-28acad2f927c │
├───────────┼──────────┼──────────────────────────────────────┤
│ trigger-2 │ 46ad5b4a │ f3675ab9-de59-49c8-a6fe-8786089f3793 │
├───────────┼──────────┼──────────────────────────────────────┤
│ trigger-3 │ 1894bff4 │ c35d61c6-961a-436f-84e0-96e0586d8635 │
├───────────┼──────────┼──────────────────────────────────────┤
│ trigger-4 │ c4b5df58 │ 6f6b0b81-6986-47ce-a075-7008b2a4eba5 │
├───────────┼──────────┼──────────────────────────────────────┤
│ trigger-5 │ ae960cee │ 9a938e7b-2ceb-4780-b5e4-011b31a68582 │
├───────────┼──────────┼──────────────────────────────────────┤
│ action-1  │ e26a7ab2 │ d4bac908-b43e-4037-8319-3306ce5dfd93 │
├───────────┼──────────┼──────────────────────────────────────┤
│ action-2  │ e343202e │ ce2fe99b-9d78-4c61-9c23-32fafa8cbf53 │
├───────────┼──────────┼──────────────────────────────────────┤
│ action-3  │ 08829b68 │ 97eae960-416f-43c3-af09-209a89278526 │
├───────────┼──────────┼──────────────────────────────────────┤
│ action-4  │ cf5e448a │ c170244d-c591-4a13-8eb5-f53e04472168 │
├───────────┼──────────┼──────────────────────────────────────┤
│ action-5  │ 5eabd0bd │ b5be0002-8078-47e5-aebc-cabaf2f69c7a │
└───────────┴──────────┴──────────────────────────────────────┘
```

リストの中から今回使用するuuidは以下です。

* trigger-1: BeagleBone BlackからDS18B20の温度データをMQTT publishするuuid
* action-1 : テスト用にMQTT subscribeするuuid
* action-2 : freeboardでWebSocket subscribeをするuuid


`trigger-1`から`action-2`へメッセージ送信を許可します。

```bash
$ docker-compose run --rm iotutil whiten -- -f trigger-1  -t action-2

> iotutil@0.0.1 start /app
> node app.js "whiten" "-f" "trigger-1" "-t" "action-2"

action-2 whitelists already contains trigger-1
```

## Pythonのプログラム

Pythonの[サンプルプログラム](https://github.com/IDCFChannel/glow-light-py)を`git clone`します。

```bash
$ cd ~/python_apps
$ git clone https://github.com/IDCFChannel/glow-light-py
$ cd glow-light-py
```

config.py.defaultをリネームしてconfig.pyを作成します。環境にあわせて上記で確認したuuidを記述します。

* IDCF_CHANNEL_URL: Meshbluコンテナが起動しているDockerホスト
* TRIGGER_UUID    : trigger-1
* ACTION_UUID     : action-1
* FREEBOARD_UUID  : action-2

```python ~/python_apps/glow-light-py/config.py
conf = {
     "IDCF_CHANNEL_URL": "210-140-160-194.jp-east.compute.idcfcloud.com",
     "TRIGGER_UUID": "c1871f89-d82e-4d49-ac43-28acad2f927c",
     "TRIGGER_TOKEN": "49f74df6",
     "ACTION_UUID": "d4bac908-b43e-4037-8319-3306ce5dfd93",
     "ACTION_TOKEN": "e26a7ab2",
     "FREEBOARD_UUID": "ce2fe99b-9d78-4c61-9c23-32fafa8cbf53"
}
```

MQTTクライアントの[Paho](https://pypi.python.org/pypi/paho-mqtt)をインストールします。

```bash
$ sudo pip install paho-mqtt
```

[BME280のプログラム](/2015/07/07/meshblu-compose-bme280-freeboard/)とほぼ同じです。`sensing()`関数で`w1_slave`の値を読みます。`28-*`は環境毎に値が異なるのでディレクトリを確認して指定します。

```python ~/python_apps/glow-light-py/ds18b20_publish.py
#!/usr/bin/python
# -*- coding: utf-8 -*-
import paho.mqtt.client as mqtt
from time import sleep
import json
import sys
from config import conf

w1="/sys/bus/w1/devices/28-0414708c9eff/w1_slave"

def sensing():
    raw = open(w1, "r").read()
    celsius = float(raw.split("t=")[-1])/1000
    retval = dict(temperature="{:.2f}".format(celsius))
    return retval

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
                               conf["FREEBOARD_UUID"],
                              "payload": retval})
             print(message)
             client.publish("message",message)
        sleep(5)

if __name__ == '__main__':
    main()
```

## Supervisor

作成したPythonスクリプトは[Supervisor](http://supervisord.org/)を使ってデモナイズします。BeagleBone BlackにSupervisorをインストールします。

```bash
$ sudo apt-get update
$ sudo apt-get install supervisor
```

1-wireを有効にするためには起動時に毎回以下のコマンドを実行する必要があります。

```bash
$ sudo sh -c 'echo BB-W1:00A0 > /sys/devices/bone_capemgr.9/slots'
```
Supervisorで実行するコマンドをシェルスクリプトにします。w1_publish.pyの前に1-wireを有効化します。


```bash ~/python_apps/glow-light-py/start_w1.sh
#!/bin/bash
sudo sh -c 'echo BB-W1:00A0 > /sys/devices/bone_capemgr.9/slots'
/home/debian/python_apps/glow-light-py/w1_publish.py
```

スクリプトに実行権限を付けます。

```bash
$ chmod +x start_w1.sh
```

DS18B20のPythonスクリプトを実行するSupervisor設定ファイルを作成します。

```bash /etc/supervisor/conf.d/w1.conf
[program:w1]
command=/home/debian/python_apps/glow-light-py/start_w1.sh
numprocs=1
redirect_stderr=true
stdout_logfile=/var/log/w1.log
user=debian
```

w1サービスを有効にします。

```bash
$ sudo supervisorctl reread
w1: available
```

w1をsupervisorのサブプロセスに追加します。

```bash
$ sudo supervisorctl add w1
w1: added process group
$ sudo supervisorctl status
w1                               RUNNING    pid 3190, uptime 0:03:51
```

## freeboard

[freeboad](https://github.com/Freeboard/freeboard)のDockerコンテナを起動します。`git clone`してある[Mesublu Compose](https://github.com/IDCFChannel/meshblu-compose)のdocker-compose.ymlにサービスが定義してあります。

```bash
$ cd ~/iot_util/meshblu-compose
```

以下が抜粋です。Dockerホストのportsは環境にあわせて変更します。

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

最初にブラウザで自己署名のSSL通信を許可します。

https://210-140-160-194.jp-east.compute.idcfcloud.com

freeboardのダッシュボードを同じ自己署名のSSL通信を許可したブラウザで開きます。

http://210-140-160-194.jp-east.compute.idcfcloud.com:8088/

freeboardがWebSocketでsubscribeするuuidは`action-2`です。MesubluのDockerホストでuuidとtokenを確認します。

```bash
$ docker-compose run --rm iotutil show -- -k action-2

> iotutil@0.0.1 start /app
> node app.js "show" "-k" "action-2"

┌──────────┬──────────┬──────────────────────────────────────┐
│ keyword  │ token    │ uuid                                 │
├──────────┼──────────┼──────────────────────────────────────┤
│ action-2 │ e343202e │ ce2fe99b-9d78-4c61-9c23-32fafa8cbf53 │
└──────────┴──────────┴──────────────────────────────────────┘
```

DATASOURCESにあるADDのリンクを押してデータソースに`action-2`のuuidとtokenを追加します。TYPEは`Octoblu`を選択します。

![freeboard-ds18b20.png](/2015/07/12/meshblu-compose-ds18b20-freeboard/freeboard-ds18b20.png)

今回は防水仕様のDS18B20を使っているので水温を計測しています。しばらくするとBeagleBone BlackからDS18B20で計測した水温が表示されます。

![freeboard-ds18b20-temperature.png](/2015/07/12/meshblu-compose-ds18b20-freeboard/freeboard-ds18b20-temperature.png)

