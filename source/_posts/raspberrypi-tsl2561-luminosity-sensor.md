title: "Raspberry Piでデジタル照度センサーのTSL2561を使う"
date: 2015-05-14 15:35:37
categories:
 - IoT
tags:
 - RaspberryPi
 - TSL2561
 - センサー
 - I2C
description: 温度や湿度のセンサーの他に手軽に扱えるセンサーに照度センサーがあります。単位はルクスで計測できます。今回はAdafruit TSL2561デジタル照度センサーを用意しました。光センサーには赤外線を使う人感センサーもあります。組み合わせて使うとIoTっぽいことができそうです。
---

温度や湿度のセンサーの他に手軽に扱えるセンサーに照度センサーがあります。単位はルクスで計測できます。今回はAdafruit [TSL2561](https://www.switch-science.com/catalog/1801/)デジタル照度センサーを用意しました。光センサーには赤外線を使う人感センサーもあります。組み合わせて使うとIoTっぽいことができそうです。

<!-- more -->

## ブレッドボード配線

ブレッドボードの配線はAdafruitのlernサイトにArduino版ですが、[TSL2561 Luminosity Sensor](https://learn.adafruit.com/tsl2561/wiring)にあります。

* GND (TSL2561) -> GPIO04 P09 (Raspberry Pi)
* VCC (TSL2561) -> 3.3v P1 (Raspberry Pi)
* SDA (TSL2561) -> GPIO02 SDA1 P03 (Raspberry Pi)
* SCL (TSL2561) -> GPIO03 SCL1 P04 (Raspberry Pi)

正常に配線ができると0x39アドレスに確認ができます。

```bash
$ i2cdetect -y 1
     0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f
00:          -- -- -- -- -- -- -- -- -- -- -- -- --
10: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
20: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
30: -- -- -- -- -- -- -- -- -- 39 -- -- -- -- -- --
40: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
50: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
60: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
70: -- -- -- -- -- -- -- --
```

## I2Cドライバ

### Python 2

I2Cバスからデータを取得するドライバは[Adafruit-Raspberry-Pi-Python-Code](https://github.com/adafruit/Adafruit-Raspberry-Pi-Python-Code)リポジトリにあるPythonの[Adafruit_I2C.py](https://github.com/adafruit/Adafruit-Raspberry-Pi-Python-Code/blob/master/Adafruit_I2C/Adafruit_I2C.py)を使います。このドライバはRaspberry PiにデフォルトでインストールされているPython 2で動作します。

``` bash
$ cd ~/python_apps
$ git clone https://github.com/adafruit/Adafruit-Raspberry-Pi-Python-Code
$ cd Adafruit-Raspberry-Pi-Python-Code/Adafruit_I2C
```

### Python 3

[Quick2Wire](https://github.com/quick2wire/quick2wire-python-api)は[RPi-Light-Sensor](https://github.com/pandringa/RPi-Light-Sensor)などでも利用されています。Python 3をRaspberry Piにインストールしていないのでvirtualenvを用意して次回使ってみようと思います。

## TSL2561プログラム

TSL2561から照度を計測するプログラムは[TSL2561 and Raspberry Pi](http://forums.adafruit.com/viewtopic.php?f=8&t=34922)のフォーラムで[TSL2561.py](https://github.com/seanbechhofer/raspberrypi/blob/master/python/TSL2561.py)が紹介されています。

また[Raspberry Pi Hacks](http://shop.oreilly.com/product/0636920029083.do)には[tsl2561-lux.py](https://github.com/spotrh/rpihacks/blob/master/tsl2561-lux.py)を使った方法が掲載されています。どちらもほぼ同じコードですが今回は[rpihacks](https://github.com/spotrh/rpihacks)リポジトリから`tsl2561-lux.py`を`Adafruit_I2C`ディレクトリにダウンロードして使います。

``` bash
$ cd ./Adafruit_I2C
$ wget https://raw.githubusercontent.com/spotrh/rpihacks/master/tsl2561-lux.py
$ chmod +x tsl2561-lux.py
```

`tsl2561-lux.py`の最後をコメントアウトしてもモジュールとして外部から呼び出せるようにします。

```py ~/python_apps/Adafruit-Raspberry-Pi-Python-Code/Adafruit_I2C/tsl2561-lux.py
...
#oLuxmeter=Luxmeter()

#print "LUX HIGH GAIN ", oLuxmeter.getLux(16)
#print "LUX LOW GAIN ", oLuxmeter.getLux(1)
#print "LUX AUTO GAIN ", oLuxmeter.getLux()
```

残念ながらファイル名にハイフンが入っているため、Pythonではこのままimportできません。アンダースコアに置換してシンボリックリンクを作ります。

```bash
$ ln -s tsl2561-lux.py tsl2561_lux.py
```

Luxmeterクラスを使って照度を計測するエントリポイントとして`tsl2561.py`を作成します。

```py ~/python_apps/Adafruit-Raspberry-Pi-Python-Code/Adafruit_I2C/tsl2561.py
#!/usr/bin/env python
# -*- coding: utf-8 -*-

from time import sleep
from tsl2561_lux import Luxmeter

if __name__ == "__main__":

    while True:
        tsl=Luxmeter()
        sensor_value = tsl.getLux()
        print(sensor_value)
        sleep(5)
```

### 実行
 
5秒間隔で照度を計測するプログラムを実行します。単位はルクスになります。瞬間の計測のためかバラツキがあるデータになりました。

``` bash
$ chmod u+x tsl2561.py
$ ./tsl2561.py
1691.71324345
961.73944068
1274.95548759
690.479479419
14.30208
0
11.50656
388.866054431
1316.01531121
0
0
```

### 暗い部屋でMQTT通信のテスト

プログラムを修正して500ルクスを超えたらMQTTにpublishするようにしました。計測間隔も3秒に短くしています。

```py:~/meshblu/publish.py
#!/usr/bin/env python
# -*- coding: utf-8 -*-

from time import sleep
from tsl2561_lux import Luxmeter
import paho.mqtt.client as mqtt
import json

def sensor():
    sensor = W1ThermSensor()
    celsius = sensor.get_temperature()
    return celsius

def on_connect(client, userdata, rc):
    print("Connected with result code " + str(rc))

def on_publish(client, userdata, mid):
    print("publish: " + str(mid))

def main():
    client = mqtt.Client(client_id='',clean_session=True,
        protocol=mqtt.MQTTv311)

    client.username_pw_set("{username}","{password}")
    client.connect("{mqtt_broker_host}", 1883, 60)

    client.on_connect = on_connect
    client.on_publish = on_publish

    tsl=Luxmeter()

    while True:
        sensor_value = tsl.getLux()
        print(sensor_value)
        if sensor_value > 500.0:
            message = json.dumps(dict(trigger="on"))
            client.publish("data",message)
        sleep(3)
if __name__ == '__main__':
    main()
```

暗い部屋に移動してテストしてみました。照度が安定して暗いときは正確に計測できています。照度があまりに大きく変わると0になるようです。iPhoneのライトをゆっくり近づけて2回publishしました。

```
$ python ./publish.py
1.42263737647
1.42263737647
1.42263737647
1.42263737647
1.42263737647
1.42263737647
1.42263737647
781.750737506
publish: 1
1212.30526902
publish: 2
2.45526898227
1.48479374557
119.459947915
287.1274
45.49168
0
0
1.45533948132
1.41892978382
1.41892978382
```
