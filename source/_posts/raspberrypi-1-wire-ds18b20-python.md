title: "Raspberry Piで1-Wireデジタル温度センサのDS18B20をPythonから使う"
date: 2015-04-12 12:54:14
categories:
 - IoT
tags:
 - RaspberryPi
 - DS18B20
 - Raspbian
 - Python
 - W1ThermSensor
 - センサー
description: いつも参考にしているAdafruitのLernサイトのAdafruit's Raspberry Pi Lesson 11. DS18B20 Temperature SensingにPythonを使ったDS18B20の温度センシングのサンプルがありました。簡単な方法としてW1ThermSensorライブラリが紹介されているのでさっそく試してみます。ライセンスはGPLv2です。
---

いつも参考にしているAdafruitのLernサイトの[Adafruit's Raspberry Pi Lesson 11. DS18B20 Temperature Sensing](https://learn.adafruit.com/adafruits-raspberry-pi-lesson-11-ds18b20-temperature-sensing)にPythonを使ったDS18B20の温度センシングのサンプルがありました。簡単な方法として[W1ThermSensor](https://github.com/timofurrer/w1thermsensor)ライブラリが紹介されているのでさっそく試してみます。ライセンスは[GPLv2 ](https://github.com/timofurrer/w1thermsensor/blob/master/setup.py)です。

<!-- more -->

## 1-Wireのセットアップ

カーネルのバージョンを確認します。

``` bash
$ uname -a
Linux raspberrypi 3.18.11+ #777 PREEMPT Sat Apr 11 17:24:23 BST 2015 armv6l GNU/Linux
```

[1-Wire in Parasite Power configuration (1-Wire using 2 wires) does not work in 3.18.3 #348](https://github.com/raspberrypi/firmware/issues/348)にあるように、カーネルが`3.18.1+`以上の場合は1-Wireを有効にする設定が変更になっています。`/boot/config.txt`に以下の行を追加して再起動します。

```txt /boot/config.txt
dtoverlay=w1-gpio-pullup,gpiopin=4
```


## w1_slaveファイルを直接読む方法

最初に`/sys/bus/w1/devices/`以下の`w1_slave`ファイルをPythonのコードから読む方法を試します。

```python ~/python_apps/w1-test.py
# -*- coding: utf-8 -*-
import os
import glob
import time
import subprocess

os.system('modprobe w1-gpio')
os.system('modprobe w1-therm')

base_dir = '/sys/bus/w1/devices/'
device_folder = glob.glob(base_dir + '28*')[0]
device_file = device_folder + '/w1_slave'

def read_temp_raw():
    catdata = subprocess.Popen(['cat',device_file], stdout=subprocess.PIPE, stderr=subprocess.PIPE)
    out,err = catdata.communicate()
    out_decode = out.decode('utf-8')
    lines = out_decode.split('\n')
    return lines

def read_temp():
    lines = read_temp_raw()
    while lines[0].strip()[-3:] != 'YES':
        time.sleep(0.2)
        lines = read_temp_raw()
    equals_pos = lines[1].find('t=')
    if equals_pos != -1:
        temp_string = lines[1][equals_pos+2:]
        temp_c = float(temp_string) / 1000.0
        temp_f = temp_c * 9.0 / 5.0 + 32.0
        return temp_c, temp_f
	
while True:
    s = "celsius: {0:.3f}, fahrenheit: {1:.3f}"
    temp = read_temp()
    print(s.format(*temp))
    time.sleep(5)
```

プログラムを実行します。摂氏と華氏を取得して小数点以下3桁でフォーマットしました。

``` bash
$ python w1-test.py
celsius: 27.312, fahrenheit: 81.162
celsius: 27.687, fahrenheit: 81.837
celsius: 27.500, fahrenheit: 81.500
```

## W1ThermSensorを使う方法

[W1ThermSensor](https://github.com/timofurrer/w1thermsensor)を`git clone`してインストールします。

``` bash
$ cd ~/python_apps
$ git clone https://github.com/timofurrer/w1thermsensor.git 
$ cd w1thermsensor
$ sudo python setup.py install
```

簡単なプログラムを書きます。センシングデータは小数点以下3桁のfloatで取得できます。

```python ~/python_apps/w1thermsensor-test.py
# -*- coding: utf-8 -*-
from w1thermsensor import W1ThermSensor

sensor = W1ThermSensor()
celsius = sensor.get_temperature()
fahrenheit = sensor.get_temperature(W1ThermSensor.DEGREES_F)
all_units = sensor.get_temperatures([W1ThermSensor.DEGREES_C, W1ThermSensor.DEGREES_F, W1ThermSensor.KELVIN])

print("celsius:    {0:.3f}".format(celsius))
print("fahrenheit: {0:.3f}".format(fahrenheit))
s = "celsius: {0:.3f}, fahrenheit: {1:.3f}, kelvin: {2:.3f}"
print(s.format(*all_units))
```

プログラムを実行します。最初の例のように`/sys/bus/w1/devices`ディレクトリから`w1_slave`ファイルを読見込む処理をライブラリが実行してくれるのでコードがシンプルになりました。

``` bash
$ python w1thermsensor-test.py
celsius:    26.625
fahrenheit: 79.925
celsius: 26.625, fahrenheit: 79.925, kelvin: 299.775
```