title: "Raspberry Pi 2で気温・気圧・高度センサのBMP180を使う"
date: 2015-06-29 11:29:40
categories:
 - IoT
tags:
 - センサー
 - BMP180
 - RaspberryPi
 - RaspberryPi2
 - スイッチサイエンス
 - SparkFun
 - Adafruit
description: 以前EdisonとArduinoでBMP180の気温・気圧・高度センサを使ってみました。Raspberry Piにはプルアップ用抵抗が最初から付いています。BMP180などI2C通信するブレイクアウトの場合はGPIOのプルアップが不要で使えます。また、AdafruitからBMP用のPythonライブラリも提供されているので初心者に優しいセンサーです。Arduino、Edison、RaspberryPiのどれでも使えるので1つあるといろいろ楽しめます。
---

以前[Edison](/2015/03/14/intel-edison-ssci-eaglet-mft-bmp180-i2c/)と[Arduino](/2015/03/21/meshblu-cylonjs-arduino-bmp180/)でBMP180の気温・気圧・高度センサを使ってみました。Raspberry Piにはプルアップ用抵抗が最初から付いています。[BMP180](https://www.switch-science.com/catalog/1598/)などI2C通信するブレイクアウトの場合はGPIOのプルアップが不要で使えます。また、AdafruitからBMP用の[Pythonライブラリ](https://github.com/adafruit/Adafruit_Python_BMP)も提供されているので初心者に優しいセンサーです。Arduino、Edison、RaspberryPiのどれでも使えるので1つあるといろいろ楽しめます。

<!-- more -->

## I2Cの設定

[Raspberry Pi 2とセンサーデータで遊ぶための初期設定](/2015/05/21/raspberrypi-2-headless-install/)に書いた方法でI2Cの設定をしています。


## 使い方

### 準備

スイッチサイエンスから[SparkFun製](https://www.sparkfun.com/products/11824)の[BMP180](https://www.switch-science.com/catalog/1598/)を購入しました。SparkFunの[配線方法](https://learn.sparkfun.com/tutorials/bmp180-barometric-pressure-sensor-hookup-)ページを見ながらつないでいきます。Pythonプログラムから使用するライブラリはAdafruitの[Adafruit_Python_BMP](https://github.com/adafruit/Adafruit_Python_BMP)を使います。


* 製品購入  : [BMP180](https://www.switch-science.com/catalog/1598/)
* 配線方法  : [BMP180 Barometric Pressure Sensor Hookup](https://learn.sparkfun.com/tutorials/bmp180-barometric-pressure-sensor-hookup-)
* ライブラリ: [Adafruit_Python_BMP](https://github.com/adafruit/Adafruit_Python_BMP)

### ブレッドボード配線

以下のように配線します。

```
DA (SDA)	 (BMP180)  -> SDA1 P02 (Raspberry Pi)
CL (SCL)	 (BMP180)  -> SCL1 P03 (Raspberry Pi)
"-" (GND)	 (BMP180)  -> GND  P09 (Raspberry Pi)
"+" (VDD)	 (BMP180)  -> 3.3v P01 (Raspberry Pi)
```

![raspi-2-bmp180.png](/2015/06/29/raspberrypi-bmp180/raspi-2-bmp180.png)

配線が終わったら[i2cdetect](http://www.lm-sensors.org/wiki/man/i2cdetect)コマンドで確認します。アドレスは0x77です。

```bash
$ i2cdetect -y 1
     0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f
00:          -- -- -- -- -- -- -- -- -- -- -- -- --
10: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
20: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
30: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
40: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
50: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
60: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
70: -- -- -- -- -- -- -- 77
```

### ライブラリのインストール

最初にPythonの開発用ライブラリをインストールします。GitHubのAdafruitリポジトリから[Adafruit_Python_BMP](https://github.com/adafruit/Adafruit_Python_BMP)をcloneします。

```bash
$ sudo apt-get update
$ sudo apt-get install git build-essential python-dev python-smbus
$ cd ~/
$ git clone https://github.com/adafruit/Adafruit_Python_BMP.git
```

sudoでPythonのパッケージをインストールします。

```bash
$ cd ~/python_apps/Adafruit_Python_BMP
$ sudo python setup.py install
...
Installed /usr/local/lib/python2.7/dist-packages/spidev-3.0-py2.7-linux-armv7l.egg
Finished processing dependencies for Adafruit-BMP==1.5.0
```

### センサーデータ取得

examplesのディレクトリに移動してサンプルのテストプログラムを実行します。BMP180だけで3種類のデータが取れました。

* 気温: 29.10 度C
* 気圧: 100,640.00 パスカル
* 高度: 57.19 メートル
* 海面気圧 : 100,635.00 パスカル

```bash
$ cd ~/python_apps/Adafruit_Python_BMP/examples
$ sudo python simpletest.py
Temp = 29.10 *C
Pressure = 100640.00 Pa
Altitude = 57.19 m
Sealevel Pressure = 100635.00 Pa
```
