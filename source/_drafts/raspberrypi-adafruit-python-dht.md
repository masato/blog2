title: "Raspberry PiでDHT11デジタル温度センサーをAdafruitライブラリから使う"
date: 2015-04-20 21:21:29
categories:
 - IoT
tags:
 - RaspberryPi
 - DHT11
 - Adafruit
 - センサー
description: 前回Adafruitのライブラリを使ってDHT11センサーを使って温度と湿度を計測しました。AdarfuitからはRaspberry Pi用にPythonライブラリも用意されています。今回ははRaspberry Piからも同様にセンシングデータを取得してみます。
---

前回Adafruitのライブラリを使ってDHT11センサーを使って温度と湿度を計測しました。AdarfuitからはRaspberry Pi用に[Pythonライブラリ](https://github.com/adafruit/Adafruit_Python_DHT)も用意されています。今回ははRaspberry Piからも同様にセンシングデータを取得してみます。

<!-- more -->

## 

 * Adafruit_Python_DHT

 https://github.com/adafruit/Adafruit_Python_DHT

* 

Raspberry Piにログインしてライブラリを`git clone`します。

``` bash
$ git clone https://github.com/adafruit/Adafruit-Raspberry-Pi-Python-Code.git
$ cd Adafruit-Raspberry-Pi-Python-Code/
$ cd Adafruit_DHT_Driver
```
##

