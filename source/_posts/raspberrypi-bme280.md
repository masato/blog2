title: "Raspberry Pi 2で温湿度・気圧センサのBME280をPythonから使う"
date: 2015-06-30 13:23:47
categories:
 - IoT
tags:
 - センサー
 - BME280
 - RaspberryPi
 - RaspberryPi2
 - スイッチサイエンス
 - Python
description: スイッチサイエンスから温湿度・気圧センサのBME280が発売されています。同じBosch製のBMP180に比べると湿度も計測できます。BME280一つあれば環境センサとして便利に使えそうです。残念なことにBoschからはCのBME280_driverしか公開されておらず、使い方のページや、The New Bosch BME280 (Temp, Humidity, BMP)のブログなどもArduinoのサンプルコードしか見つかりませんでした。Raspberry  Pi用にPythonで書き換えるのは敷居が高くて困っていると、ありがたいことにスイッチサイエンスさんが速攻でPythonのライブラリを書いてくれました。
---

スイッチサイエンスから温湿度・気圧センサのBME280が発売されています。同じBosch製の[BMP180](/2015/06/29/raspberrypi-bmp180/)に比べると湿度も計測できます。BME280一つあれば環境センサとして便利に使えそうです。残念なことにBoschからはCの[BME280_driver](https://github.com/BoschSensortec/BME280_driver)しか公開されておらず、[使い方](http://trac.switch-science.com/wiki/BME280)のページや、[The New Bosch BME280 (Temp, Humidity, BMP)](http://arduinotronics.blogspot.jp/2015/05/the-new-bosch-bme280-temp-humidity-bmp.html)のブログなどもArduinoのサンプルコードしか見つかりませんでした。Raspberry  Pi用にPythonで書き換えるのは敷居が高くて困っていると、ありがたいことにスイッチサイエンスさんが速攻でPythonのライブラリを書いてくれました。

<!-- more -->

## ブレッドボード配線

[使い方](http://trac.switch-science.com/wiki/BME280)にあるArduinoの回路図をみながら、Raspberry Pi 2に以下のように配線します。確認したところ3.3Vに配線するVcoreとVioは基板上でつながっているのでどちらか片方だけでよいみたいです。

```
SDI      (BME280)  -> GPIO2 P03 (Raspberry Pi SDA1)
SCK      (BME280)  -> GPIO3 P05 (Raspberry Pi SCL1)
GND,SDO  (BME280)  -> GND  P09 (Raspberry Pi)
Vio,CSB  (BME280)  -> 3.3v P01 (Raspberry Pi)
```

![bme280-plus.png](/2015/06/30/raspberrypi-bme280/bme280-plus.png)

ジャンパワイヤを配線したらi2cdetectで確認します。0x76アドレスを使っています。

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

## サンプルコード

スイッチサイエンスの[BME280](https://github.com/SWITCHSCIENCE/BME280)のリポジトリにPythonのサンプルコードがあります。適当なディレクトリを作成してダウンロードします。


```bash
$ mkdir ~/python_apps/ss
$ cd !$
$ wget https://raw.githubusercontent.com/SWITCHSCIENCE/BME280/master/Python27/bme280_sample.py
```

サンプルコードを実行します。

```bash
$ sudo python bme280_sample.py
temp : 28.18  ℃
pressure :  995.69 hPa
hum :  60.62 ％
```

以下のデータが取得できました。

* 気温: 28.18 度C
* 気圧: 995.69 ヘクトパスカル
* 湿度: 60.62 ％