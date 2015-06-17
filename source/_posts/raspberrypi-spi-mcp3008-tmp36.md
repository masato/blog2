title: "Raspberry PiのPythonからTMP36のアナログ温度センサとMCP3008のADコンバータを使う"
date: 2015-04-11 14:15:56
categories:
 - IoT
tags:
 - RaspberryPi
 - TMP36
 - MCP3008
 - センサー
 - Python
description: Raspberry Piからセンシングデータを取得する場合もArduino Firmataと同様にNode.jsから行いたかったのですが難しそうなのでPythonに戻ることにしました。Raspberry PiのGPIOはデジタル入力しかできないのでTMP36やLM35DZのアナログセンサを使う場合はMCP3008やPCF8591などのADコンバータが必要になります。TMP36を使い温度を計測するPythonプログラムを書いてみます。
---

Raspberry Piからセンシングデータを取得する場合もArduino Firmataと同様にNode.jsから行いたかったのですが難しそうなのでPythonに戻ることにしました。Raspberry PiのGPIOはデジタル入力しかできないので[TMP36](http://eleshop.jp/shop/g/gBB2122/)や[LM35DZ](http://eleshop.jp/shop/g/g731131/)のアナログセンサを使う場合は[MCP3008](https://www.switch-science.com/catalog/1514/)や[PCF8591](http://www.aitendo.com/product/10190)などのADコンバータが必要になります。TMP36を使い温度を計測するPythonプログラムを書いてみます。

<!-- more -->

## SPIを有効にする

[SPI(Serial Peripheral Interface)](http://ja.wikipedia.org/wiki/%E3%82%B7%E3%83%AA%E3%82%A2%E3%83%AB%E3%83%BB%E3%83%9A%E3%83%AA%E3%83%95%E3%82%A7%E3%83%A9%E3%83%AB%E3%83%BB%E3%82%A4%E3%83%B3%E3%82%BF%E3%83%95%E3%82%A7%E3%83%BC%E3%82%B9)はシリアル通信の規格の一つです。Raspberry PiのSPIを有効化する方法がカーネルの3.18から変更になったようです。[Enabling The SPI Interface On The Raspberry Pi](http://www.raspberrypi-spy.co.uk/2014/08/enabling-the-spi-interface-on-the-raspberry-pi/)にraspi-configを使う方法と手動で有効にする方法が書いてあります。今回は手動で行います。

### ファームウェアを更新する

カーネルが3.18.xでない場合はファームウェアを最新します。あわせてパッケージも最新にしました。

``` bash
$ sudo apt-get update
$ sudo apt-get upgrade
$ sudo rpi-update
```

カーネルのバージョンを確認します。

``` bash
$ cat /proc/version
Linux version 3.18.11+ (dc4@dc4-XPS13-9333) (gcc version 4.8.3 20140303 (prerelease) (crosstool-NG linaro-1.13.1+bzr2650 - Linaro GCC 2014.03) ) #777 PREEMPT Sat Apr 11 17:24:23 BST 2015
```

`/boot/config.txt`を編集します。

``` bash
$ sudo vi /boot/config.txt
dtparam=spi=on
```

rebootします。

``` bash
$ sudo reboot
```

SPIが有効になりました。

``` bash
$ lsmod | grep spi
spi_bcm2708             6018  0
```

## ブレッドボード配線

MCP3008とRaspberry PiとFT232RLをブレッドボードで配線します。

![mcp3008-tmp36.png](/2015/04/11/raspberrypi-spi-mcp3008-tmp36/mcp3008-tmp36.png)

MCP3008とRaspberry Piと配線します。

* MCP3008 VDD     -> Raspberry Pi 3.3V
* MCP3008 VREF    -> Raspberry Pi 3.3V
* MCP3008 AGND    -> Raspberry Pi GND
* MCP3008 CLK     -> Raspberry Pi GPIO 11 (SCLK)
* MCP3008 DOUT    -> Raspberry Pi GPIO 9 (MISO)
* MCP3008 DIN     -> Raspberry Pi GPIO 10 (MOSI)
* MCP3008 CS/SHDN -> Raspberry Pi GPIP 8 (CE0)
* MCP3008 DGND    -> Raspberry Pi GND

TMP36を配線します。

* TMP36 VCC     -> Raspberry Pi 3.3V
* TMP36 VOUT    -> MCP3008 CH0
* TMP36 GND     -> Raspberry Pi GND

## Pythonのインストール

Raspberry PiのPython開発環境をインストールします。

``` bash
$ sudo apt-get update
$ sudo apt-get install python-dev
$ python -V
Python 2.7.3
```

ez_setupとpipをインストールします。

``` bash
$ curl https://bootstrap.pypa.io/ez_setup.py -o - | sudo python
$ curl https://bootstrap.pypa.io/get-pip.py -o - | sudo python
$ pip -V
pip 6.1.1 from /usr/local/lib/python2.7/dist-packages (python 2.7)
```


### py-spidev

PythonからSPIデバイスを操作するため[py-spidev](https://github.com/doceme/py-spidev)をインストールします。

``` bash
$ cd
$ git clone git://github.com/doceme/py-spidev 
$ cd py-spidev
$ sudo python setup.py install
...
Writing /usr/local/lib/python2.7/dist-packages/spidev-3.0.egg-info
```

### サンプルプロジェクト

プロジェクトを作成します。

``` bash
$ mkdir -p ~/python_apps/spidev-spike
$ cd !$
```

以下のサイトを参考にしてPythonのサンプルプログラムを書きます。

* [ラズパイでセンサーのデータを継続的に記録する（ソフトウェア編）](http://windvoice.hatenablog.jp/entry/2015/03/04/165820)
* [Analogue Sensors On The Raspberry Pi Using An MCP3008](http://www.raspberrypi-spy.co.uk/2013/10/analogue-sensors-on-the-raspberry-pi-using-an-mcp3008/)
* [Serial Peripheral Interface](http://raspberrypi-aa.github.io/session3/spi.html)

```python ~/python_apps/spidev-spike/spi_tmp36.py
#!/usr/bin/env python

import time
import sys
import spidev

spi = spidev.SpiDev()
spi.open(0,0)

def readAdc(channel):
    adc = spi.xfer2([1,(8+channel)<<4,0])
    data = ((adc[1]&3) << 8) + adc[2]
    return data

def convertVolts(data):
    volts = (data * 3.3) / float(1023)
    volts = round(volts,4)
    return volts

def convertTemp(volts):
    temp = (100 * volts) - 50.0
    temp = round(temp,4)
    return temp

if __name__ == '__main__':
    try:
        while True:
            data = readAdc(0)
            print("adc  : {:8} ".format(data))
            volts = convertVolts(data)
            temp = convertTemp(volts)
            print("volts: {:8.2f}".format(volts))
            print("temp : {:8.2f}".format(temp))

            time.sleep(5)
    except KeyboardInterrupt:
        spi.close()
        sys.exit(0)
```

### プログラム実行

Pythonのプログラムを実行します。5秒間隔で気温を摂氏で表示します。

``` bash
$ python spi_tmp36.py
adc  :      232
volts:     0.75
temp :    24.84
adc  :      233
volts:     0.75
temp :    25.16
adc  :      233
volts:     0.75
temp :    25.16
```


