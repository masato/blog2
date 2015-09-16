title: "Raspberry Pi 2からBME280環境センサのデータをPythonでMQTT publishする"
date: 2015-07-01 15:35:39
categories:
 - IoT
tags:
 - センサー
 - BME280
 - RaspberryPi
 - RaspberryPi2
 - Python
 - Meshblu
 - MQTT
 - Paho
description: Raspberry Pi 2のPythonを使いスイッチサイエンスが販売しているBME280の環境センサからデータを取れるようになりました。の環境センサのデータを取れるようになりました。次はPahoのMQTTクライアントをインストールしてpublishしてみます。
---

[Raspberry Pi 2のPythonを使い](/2015/06/30/raspberrypi-bme280/)スイッチサイエンスが販売している[BME280](https://www.switch-science.com/catalog/2236/)の環境センサからデータを取れるようになりました。の環境センサのデータを取れるようになりました。次は[Paho](http://www.eclipse.org/paho/)の[MQTTクライアント](https://pypi.python.org/pypi/paho-mqtt)をインストールしてpublishしてみます。
<!-- more -->

## プロジェクト

Raspberry Pi 2は本格的なプログラミングを書くのは不向きですが、この程度のコードでしたら直接vimで書いてしまいます。Raspberry Pi 2にログインして適当なディレクトリにプロジェクトを作成します。今回作成したリポジトリは[こちら](https://github.com/masato/bme280-meshblu-py)です。最終的にディレクトリは以下のようになります。

```bash
$ cd ~/python_apps/bme280
$ tree
.
├── bme280_publish.py
├── bme280.py
├── config.py
└── README.md
```

[Pythonライブラリ](https://github.com/SWITCHSCIENCE/BME280/tree/master/Python27)をスイッチサイエンスのリポジトリからダウンロードして使います。

```bash
$ cd ~/python_apps/bme280
$ wget https://raw.githubusercontent.com/SWITCHSCIENCE/BME280/master/Python27/bme280_sample.py
```


## MQTT

### MeshbluのMQTTブローカー

私の場合[Meshblu](https://github.com/octoblu/meshblu)のMQTTブローカーを使っています。MeshbluはIoT向けのメッセージングプラットフォームです。MQTTの他にHTTP REST、WebSocket、CoAPなど複数のプロトコルに対応し、相互にブリッジが可能です。MeshbluはNode.jsで書かれているので、MQTTブローカーの実装もNode.jsの[Mosca](https://github.com/mcollina/mosca)を使っています。今回はMeshbluを使いますが、オープンソースのMQTTブローカーの実装には[Mosca](https://github.com/mcollina/mosca)、[Mosquitto](http://mosquitto.org/)、[Ponte](https://eclipse.org/ponte/)などいろいろあります。

またMeshbluのJavaScriptクライアントを使うと、データソースとして対応している[freeboard](https://github.com/Freeboard/freeboard)では直接ブラウザから、MQTTでpublishされたメッセージをWebScocket経由でsubscribeすることもできます。

### PahoのMQTTクライアント

MQTTクライアントは[Paho](http://www.eclipse.org/paho/)の[paho-mqtt](https://pypi.python.org/pypi/paho-mqtt)を使います。

```bash
$ pip install paho-mqtt
```

MQTTブローカーの設定は簡単にPythonのディクショナリに書いてimportしました。

```python ~/python_apps/bme280-meshblu-py/config.py
conf = {
     "MESHBLU_URL": "xxx.xxx.xxx.xxx",
     "MESHBLU_USER": "5abcfad1-9129-4f4f-b946-cabb6ecd9f6a",
     "MESHBLU_PASSWORD": "d8b721ed",
     "SEND_TO": "28cbe216-1c1c-477a-bbd5-5ee81d30ba02"
}
```

## サンプルコード

[bme280_sample.py]()はBME280から取得した環境データを標準出力するサンプルです。MQTTクライアントのプログラムからimportして使うように少しプログラムを修正します。

### bme280.py

bme280_sample.pyをコピーしてbme280.pyを作成して編集します。[修正箇所](https://github.com/masato/bme280-meshblu-py/blob/master/bme280_sample.diff)は以下です。関数内で標準出力をしているところをreturnして、エントリポイントでJSONにして返すだけです。

```python
@@ -66,9 +66,12 @@
 	temp_raw = (data[3] << 12) | (data[4] << 4) | (data[5] >> 4)
 	hum_raw  = (data[6] << 8)  |  data[7]
 	
-	compensate_T(temp_raw)
-	compensate_P(pres_raw)
-	compensate_H(hum_raw)
+	temperature = compensate_T(temp_raw)
+	pressure = compensate_P(pres_raw)
+	humidity = compensate_H(hum_raw)
+        return dict(temperature=temperature,
+                    pressure=pressure,
+                    humidity=humidity)
 
 def compensate_P(adc_P):
 	global  t_fine
@@ -92,7 +95,8 @@
 	v2 = ((pressure / 4.0) * digP[7]) / 8192.0
 	pressure = pressure + ((v1 + v2 + digP[6]) / 16.0)  
 
-	print "pressure : %7.2f hPa" % (pressure/100)
+	#print "pressure : %7.2f hPa" % (pressure/100)
+        return "{:.2f}".format(pressure/100)
 
 def compensate_T(adc_T):
 	global t_fine
@@ -100,7 +104,8 @@
 	v2 = (adc_T / 131072.0 - digT[0] / 8192.0) * (adc_T / 131072.0 - digT[0] / 8192.0) * digT[2]
 	t_fine = v1 + v2
 	temperature = t_fine / 5120.0
-	print "temp : %-6.2f ℃" % (temperature) 
+	#print "temp : %-6.2f ℃" % (temperature) 
+        return "{:.2f}".format(temperature)
 
 def compensate_H(adc_H):
 	global t_fine
@@ -114,8 +119,8 @@
 		var_h = 100.0
 	elif var_h < 0.0:
 		var_h = 0.0
-	print "hum : %6.2f ％" % (var_h)
-
+	#print "hum : %6.2f ％" % (var_h)
+        return "{:.2f}".format(var_h)
 
 def setup():
 	osrs_t = 1			#Temperature oversampling x 1
```

### bme280_publish.py

MQTTクライアントを使うメインプログラムを実装します。スイッチサイエンスさんのbme280_sample.pyをライブラリ用に修正したbme280と、paho-mqttをimportして使います。5秒間隔で環境データを計測してJSONにフォーマット後、MQTTブローカーにpublishするだけです。


```python ~/python_apps/bme280-meshblu-py/bme280_publish.py
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

    client.username_pw_set(conf["MESHBLU_USER"], conf["MESHBLU_PASSWORD"])
    client.on_connect = on_connect
    client.on_publish = on_publish
    client.connect(conf["MESHBLU_URL"], 1883, 60)

    while True:
        retval = sensing()
        if retval:
             message = json.dumps({"devices":
	                      conf["SEND_TO"],
                              "payload": retval})
             print(message)
             client.publish("message",message)
        sleep(5)

if __name__ == '__main__':
    main()
```

今回はMeshbluをMQTTブローカーに使っているため、messageのフォーマットが決まっています。他のMQTTブローカーを使うときはpayloadだけで構いません。また`devices`のキーに宛先を指定して、topic名は`message`を固定で使う仕様になっています。

```JSON
{"payload": {"pressure": "999.56", "temperature": "28.94", "humidity": "59.14"}, 
 "devices": "28cbe216-1c1c-477a-bbd5-5ee81d30ba02"}
```

## テスト

### MQTT subscribe

MQTTのクライアントは、MQTTブローカーのホストにMosquittoのクライアントをインストールします。

```bash
$ sudo apt-get install mosquitto_client
```

mosquitto_subコマンドを使ってsubscribeします。こちらもMeshbluの仕様なのでtopic名やユーザー名はMQTTブローカーの仕様に合わせて使います。

```bash
$ mosquitto_sub \
  -h localhost \
  -p 1883 \
  -t 28cbe216-1c1c-477a-bbd5-5ee81d30ba02 \
  -u 28cbe216-1c1c-477a-bbd5-5ee81d30ba02 \
  -P 9e7cbe84 \
  -d
Received CONNACK
Received SUBACK
Subscribed (mid: 1): 0
```

### MQTT publish

Raspberry Pi 2に戻り、作成したbme280_publish.pyをsudoで実行します。piユーザーをi2cグループに追加している場合はsudoは不要です。

```bash
$ sudo ./bme280_publish.py
{"payload": {"pressure": "999.96", "temperature": "28.79", "humidity": "58.76"}, "devices": "28cbe216-1c1c-477a-bbd5-5ee81d30ba02"}
publish: 1
{"payload": {"pressure": "999.97", "temperature": "28.74", "humidity": "58.07"}, "devices": "28cbe216-1c1c-477a-bbd5-5ee81d30ba02"}
publish: 2
```

5秒間隔でBME280から取得した環境データをMQTTブローカーにpublishし始めました。

再びMQTTブローカーのホストに戻ると、mosquitto_subコマンドを実行しているシェルにメッセージが標準出力されました。

```bash
...
Subscribed (mid: 1): 0
Received PUBLISH (d0, q0, r0, m0, '28cbe216-1c1c-477a-bbd5-5ee81d30ba02', ... (71 bytes))
{"data":{"pressure":"999.97","temperature":"28.74","humidity":"58.07"}}
Received PUBLISH (d0, q0, r0, m0, '28cbe216-1c1c-477a-bbd5-5ee81d30ba02', ... (71 bytes))
{"data":{"pressure":"999.97","temperature":"28.74","humidity":"58.07"}}
```