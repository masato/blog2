title: "Raspberry PiでPythonのRPi.GPIOを使ってLチカする"
date: 2015-04-15 11:01:59
categories:
 - IoT
tags:
 - RaspberryPi
 - Python
 - Lチカ
description: 以前Raspberry PiでNode.jsのJohnny-Fiveを使ったLチカのサンプルを書きました。今回はPythonのRPi.GPIOパッケージをインストールしてLチカしてみます。
---

以前Raspberry PiでNode.jsの[Johnny-Five](2015/03/31/raspberrypi-raspi-io-wiringpi-led-blinking/)を使ったLチカのサンプルを書きました。今回はPythonの[RPi.GPIO](https://pypi.python.org/pypi/RPi.GPIO)パッケージをインストールしてLチカしてみます。

<!-- more -->

## RPi.GPIO 

[RPi.GPIO](https://pypi.python.org/pypi/RPi.GPIO)パッケージはRaspberry PiのGPIOを操作するときに使います。pipやapt-getからインストールできます。

pipの場合

``` bash
$ sudo pip install rpi.gpio
```

apt-getの場合

``` bash
$ sudo apt-get install python-rpi.gpio
```

### GPIO.BOARDとGPIO.BCM

RPi.GPIOのサンプルを見ているとGPIOのピン番号を指定する方法が2種類あります。何が違うのか調べていると[What is the difference between BOARD and BCM for GPIO pin numbering?](http://raspberrypi.stackexchange.com/questions/12966/what-is-the-difference-between-board-and-bcm-for-gpio-pin-numbering)に説明がありました。

* GPIO.BOARD: PIN番号
* GPIO.BCM: GPIO番号

今回LEDのアノードを配線する`PIN 11`は、Raspberry Pi Model Bの場合`GPIO 17`になります。GPIO番号はモデルごと違う場合があるので、PIN番号で指定するGPIO.BOARDを使った方がよいみたいです。


### This channel is already in use

RPi.GPIOのプログラムを実行すると`This channel is already in use`の警告がでる場合があります。これは以前`GPIO.setup()`を実行してポートのセットアップをした後、クリアしていないときに発生します。

```bash
$ sudo python led-blink.py
led-blink.py:8: RuntimeWarning: This channel is already in use, continuing anyway.  Use GPIO.setwarnings(False) to disable warnings.
  GPIO.setup(PIN,GPIO.OUT)
```

プログラムの最後にポートのクリアを行います。

``` python
GPIO.cleanup()
```

## ブレッドボード配線

抵抗器は1K(茶黒赤)を用意しました。LEDを配線します。

![raspi-led.png](/2015/04/15/raspberrypi-led-blink/raspi-led.png)

## サンプルプログラム

```python ~/python_apps/led-blink.py
# -*- coding: utf-8 -*-
import RPi.GPIO as GPIO
import time

COUNT = 3
PIN = 11
GPIO.setmode(GPIO.BOARD)
GPIO.setup(PIN,GPIO.OUT)

for _ in xrange(COUNT):
    GPIO.output(PIN,True)
    time.sleep(1.0)
    GPIO.output(PIN,False)
    time.sleep(1.0)

GPIO.cleanup()
```

GPIOを操作する場合はroot権限が必要になります。プログラムを実行すると3回、1秒間隔でLチカします。

``` bash
$ sudo python led-blink.py
```


