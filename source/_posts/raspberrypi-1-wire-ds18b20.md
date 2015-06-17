title: "Raspberry Piで1-Wireデジタル温度センサのDS18B20を使う"
date: 2015-04-01 12:16:11
tags:
 - RaspberryPi
 - DS18B20
 - Raspbian
 - センサー
description: Raspberry PiのGPIOはデジタルなのでArduinoで使っていたTMP36やLM35DZなどのアナログセンサを使う場合はADコンバータが必要になります。今回は1-Wireデジタル温度センサのDS18B20を使ってみます。
---

Raspberry PiのGPIOはデジタルなのでArduinoで使っていたTMP36やLM35DZなどのアナログセンサを使う場合はADコンバータが必要になります。今回は1-Wireデジタル温度センサのDS18B20を使ってみます。

<!-- more -->

## ブレッドボード配線

DS18B20を搭載したセンサモジュールはaitendoから[1820-3PL](http://www.aitendo.com/product/10812)を395円で購入しました。こちらは完成品のモジュールになっています。

* `+` (DS18B20) -> 3.3V  (Raspberry Pi)
* `S` (DS18B20) -> GPIO4 (Raspberry Pi)
* `-` (DS18B20) -> GND (Raspberry Pi)

## セットアップ

最初にパッケージを最新にしてアップグレードします。

``` bash
$ sudo apt-get update
$ sudo apt-get upgrade
```

カーネルのバージョンを確認します。

``` bash
$ uname -a
Linux raspberrypi 3.18.7+ #755 PREEMPT Thu Feb 12 17:14:31 GMT 2015 armv6l GNU/Linux
```

[Raspberry Piで遊ぼう! 改訂第3版 最新情報お知らせページ](http://v7.com/raspi3/?page=3)に以下のような情報がありました。

> Raspbian 2015-01-31（NOOBS 1.3.12）以降では、1-wireを有効にするための設定手順が増えました。

`/boot/config.txt`の書き方が変更になったようです。以下の行をファイルの最後に追記します。

```txt /boot/config.txt
dtoverlay=w1-gpio-pullup,gpiopin=4
```

再起動します。

``` bash
$ sudo reboot
```

w1-gpioとw1-thermのカーネルモジュールをロードします。

``` bash
$ sudo modprobe w1-gpio
$ sudo modprobe w1-therm
```

## 使い方

カーネルモジュールをロードすると`/sys/bus/w1/devices`ディレクトリが作成されます。

``` bash
$ cd /sys/bus/w1/devices
$ ls
28-0314626a2cff  w1_bus_master1
```

'28-'で始まる値は温度センサーのデバイスIDです。この値はデバイス毎に異なります。'28-'のディレクトリに移動して`w1_slave`ファイルを読みます。


``` bash
$ cd 28-0314626a2cff 
$ cat w1_slave
b3 01 55 00 7f ff 0c 10 63 : crc=63 YES
b3 01 55 00 7f ff 0c 10 63 t=27187
$ cat w1_slave
b4 01 55 00 7f ff 0c 10 b3 : crc=b3 YES
b4 01 55 00 7f ff 0c 10 b3 t=27250
```

1行目にYESと表示されていると温度の計測が成功しています。`t=27187`の数字を1000分の1をすると摂氏になります。摂氏27.187度と摂氏27.250度が計測できました。
