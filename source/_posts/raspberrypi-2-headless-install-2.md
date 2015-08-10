title: "myThingsをはじめよう - Part1: はじめてのRaspberry Pi 2セットアップ"
date: 2015-07-16 11:36:31
categories:
 - IoT
tags:
 - RaspberryPi2
 - BME280
 - Lチカ
 - I2C
 - スイッチサイエンス
 - IDCFクラウド
description: Raspberry Piにはじめて興味を持った方でも、インターネットへの接続、Lチカと環境センサからデータを取得できるようになるまでの手順をまとめてみようと思います。
---

Raspberry Piにはじめて興味を持った方でも、インターネットへの接続、Lチカと環境センサからデータを取得できるようになるまでの手順をまとめてみようと思います。

<!-- more -->

## myThingsをはじめようキット

今回はスイッチサイエンスから発売された[myThingsをはじめようキット](https://www.switch-science.com/catalog/2366/)を使います。このスターターキットを使うと[IDCFクラウド](http://www.idcf.jp/cloud/iot/)を経由して、自作したIoT機器を[myThings](http://mythings.yahoo.co.jp/)と連係できるようになります。はじめてのRaspberry Piの入門キットとして必要な部品が集まっています。

### 用意するもの

Raspberry Piなどをすでに持っている場合は、個別に必要な部品を用意します。Lチカだけだと物足りないので、気温、湿度、気圧が計測できる環境センサの[BME280](https://www.switch-science.com/catalog/2236/)もあわせて使います。

* [Raspberry Pi 2 Model B（RSコンポーネンツ製）](https://www.switch-science.com/catalog/2127/)
* [Raspberry Pi用microSD 8GB（Raspbian OS 書き込み済）](https://www.switch-science.com/catalog/2252/)
* [FTDI USBシリアル変換アダプター(5V/3.3V切り替え機能付き)](https://www.switch-science.com/catalog/1032/)
* [PLANEX GW-USNANO2A 無線LAN USBアダプタ](https://www.switch-science.com/catalog/2115/)
* [ラズパイで作ろう！ゼロから学ぶロボット製作教室電子部品セット その１（連載第2回向け）](https://www.switch-science.com/catalog/2250/)
* [固いジャンパワイヤ　(ブレッドボード用)](https://www.switch-science.com/catalog/314/)
* [BME280搭載　温湿度・気圧センサモジュール](https://www.switch-science.com/catalog/2236/)


## USBシリアル変換アダプター

[FTDI USBシリアル変換アダプター(5V/3.3V切り替え機能付き)](https://www.switch-science.com/catalog/1032/)を使ってOSXとRaspberry Pi 2を接続します。[Software Installation (Mac)]( https://learn.adafruit.com/adafruits-raspberry-pi-lesson-5-using-a-console-cable?view=all)を参考にしてGPIOピンにつなぎます。電圧切り換えジャンパは3.3V側にしてください。電源はUSB ACアダプタから供給するのでこのUSBシリアル変換アダプタから電源供給はしないでください。

* GND 黒 (USB-TTL)  ->  GND P6  (Raspberry Pi)
* RXD 黄色 (USB-TTL)  ->  TXD GPIO14 P8  (Raspberry Pi)
* TXD 緑 (USB-TTL)  ->  RXD GPIO15 P10 (Raspberry Pi)

![raspi-ftdi2.png](/2015/07/16/raspberrypi-2-headless-install-2/raspi-ftdi2.png)

FTDIのFT232RL用のドライバは[VCP Drivers](http://www.ftdichip.com/Drivers/VCP.htm)からダウンロードしてインストールします。今回のホストマシンはOSX Yosemite 10.10.4です。`Mac OS X 10.9 and above`の[FTDIUSBSerialDriver_v2_3.dmg](http://www.ftdichip.com/Drivers/VCP/MacOSX/FTDIUSBSerialDriver_v2_3.dmg)をインストールします。`Mac OS X 10.3 to 10.8`の場合は、[FTDIUSBSerialDriver_v2_2_18.dmg](http://www.ftdichip.com/Drivers/VCP/MacOSX/FTDIUSBSerialDriver_v2_2_18.dmg)を使います。

OSXからScreenを使ってターミナル接続します。`Axxxx`の箇所は環境によって異なります。

``` bash
$ screen /dev/tty.usbserial-Axxxx 115200
```

## 無線LAN USBアダプタ

無線LAN USBアダプタは[PLANEX GW-USNANO2A](https://www.switch-science.com/catalog/2115/)を使います。wpa_passphraseコマンドから無線LANの設定を入れます。

``` bash
$ sudo sh -c 'wpa_passphrase {ssid} {passphrase} >> /etc/wpa_supplicant/wpa_supplicant.conf'
```

今回の無線LAN環境はSSIDステルスモードなので`/etc/wpa_supplicant/wpa_supplicant.conf`を編集して`scan_ssid=1`を追加します。また複数のネットワーク設定を記述する場合は、`priority`を追加します。デフォルトは0で、値が大きい方が優先されます。

```bash /etc/wpa_supplicant/wpa_supplicant.conf
ctrl_interface=DIR=/var/run/wpa_supplicant GROUP=netdev
update_config=1
network={
        priority=0
        scan_ssid=1
        ssid="xxx"
        proto=WPA2
        key_mgmt=WPA-PSK
        #psk="xxx"
        psk=xxx
}
```

`/etc/network/interfaces`を編集します。今回はIPアドレスはDHCPから取得するため、`iface wlan0 inet manual`を`iface wlan0 inet dhcp`また`wpa-conf`になっていることを確認します。今回は`wlan1`は使いませんがこちらもdhcpに変更しておきます。

```bash /etc/network/interfaces
auto lo
iface lo inet loopback

auto eth0
allow-hotplug eth0
iface eth0 inet manual

auto wlan0
allow-hotplug wlan0
#iface wlan0 inet manual
iface wlan0 inet dhcp
wpa-conf /etc/wpa_supplicant/wpa_supplicant.conf

auto wlan1
allow-hotplug wlan1
#iface wlan1 inet manual
iface wlan1 inet dhcp
wpa-conf /etc/wpa_supplicant/wpa_supplicant.conf
```

wlan0を再起動します。

```bash
$ sudo ifdown wlan0
$ sudo ifup wlan0
```

DHCPからIPアドレスが取得できたら、pingでインターネットに接続できることを確認します。

```bash
$ ip addr show wlan0
$ ping -c 1 www.yahoo.co.jp
PING www.g.yahoo.co.jp (183.79.198.116) 56(84) bytes of data.
64 bytes from f9.top.vip.kks.yahoo.co.jp (183.79.198.116): icmp_req=1 ttl=50 time=20.8 ms

--- www.g.yahoo.co.jp ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 20.884/20.884/20.884/0.000 ms
```

## raspi-configの初期設定

raspi-configをroot権限で実行して、Raspberry Pi 2の初期設定をしていきます。

```bash
$ sudo raspi-config
```

ファイルシステムの拡張とタイムゾーンの変更を行います。

* Expand Filesystem > Select
* Internationalisation Options > Change Timezone > Asia > Tokyo

再起動します。

``` bash
$ sudo reboot
```

再起動後にファイルシステムがリサイズされました。ようやくパッケージをインストールする準備ができました。

```bash
$ df -h
Filesystem      Size  Used Avail Use% Mounted on
rootfs          7.3G  2.4G  4.6G  35% /
/dev/root       7.3G  2.4G  4.6G  35% /
devtmpfs        460M     0  460M   0% /dev
tmpfs            93M  240K   93M   1% /run
tmpfs           5.0M     0  5.0M   0% /run/lock
tmpfs           186M     0  186M   0% /run/shm
/dev/mmcblk0p1   56M   19M   37M  34% /boot
```


## ファームウェアの更新

### 初期状態の確認

今回利用している[Raspberry Pi用microSD 8GB（Raspbian OS 書き込み済）](https://www.switch-science.com/catalog/2252/)のバージョンを確認します。

```bash
$ uname -a
Linux raspberrypi 3.18.11-v7+ #781 SMP PREEMPT Tue Apr 21 18:07:59 BST 2015 armv7l GNU/Linux
$ cat /proc/version
Linux version 3.18.11-v7+ (dc4@dc4-XPS13-9333) (gcc version 4.8.3 20140303 (prerelease) (crosstool-NG linaro-1.13.1+bzr2650 - Linaro GCC 2014.03) ) #781 SMP PREEMPT Tue Apr 21 18:07:59 BST 2015
```

Raspbian OSは`3.18.11-v7+`が入っていました。つぎにpiユーザーのグループを確認すると、i2cグループに最初から入っています。piユーザーはsudoなしでi2cデバイスを操作できるようになっています。


``` bash
$ id -a
uid=1000(pi) gid=1000(pi) groups=1000(pi),4(adm),20(dialout),24(cdrom),27(sudo),29(audio),44(video),46(plugdev),60(games),100(users),106(netdev),996(gpio),997(i2c),998(spi),999(input)
```

パッケージの更新とアップグレードをします。

``` bash
$ sudo apt-get update
$ sudo apt-get upgrade
```

ファームウェアの更新もしておきます。

``` bash
$ sudo rpi-update
...
 *** A reboot is needed to activate the new firmware
```

rebootします。

``` bash
$ sudo reboot
```

カーネルのバージョン、RaspbianOSのバージョンが`4.0.8-v7+`にあがりました。

``` bash
$ uname -a
Linux raspberrypi 4.0.8-v7+ #805 SMP PREEMPT Thu Jul 16 18:46:20 BST 2015 armv7l GNU/Linux
$ cat /proc/version
Linux version 4.0.8-v7+ (dc4@dc4-XPS13-9333) (gcc version 4.8.3 20140303 (prerelease) (crosstool-NG linaro-1.13.1+bzr2650 - Linaro GCC 2014.03) ) #805 SMP PREEMPT Thu Jul 16 18:46:20 BST 2015
```

## その他設定

### vim

エディタは好みですがvimを使います。

``` bash
$ sudo apt-get install vim
$ mkdir ~/tmp
```

バックアップを作らない設定だけします。

``` bash ~/.vimrc
set backupdir=~/tmp
```

### 時間合わせ

NTPサーバーを日本のサーバーに設定します。

``` bash /etc/ntp.conf
#server 0.debian.pool.ntp.org iburst
#server 1.debian.pool.ntp.org iburst
#server 2.debian.pool.ntp.org iburst
#server 3.debian.pool.ntp.org iburst
pool ntp.nict.jp iburst
```

ntpdの再起動をして時間あわせをします。

``` bash
$ sudo /etc/init.d/ntp restart
Stopping NTP server: ntpd.
Starting NTP server: ntpd.
```

## Python

Raspberry Pi 2でセンサーデータを取得する場合はPythonを使うことが多いです。Pythonの開発環境をインストールします。

``` bash
$ sudo apt-get update
$ sudo apt-get install python-dev
$ python -V
Python 2.7.3
$ curl https://bootstrap.pypa.io/get-pip.py -o - | sudo python
$ pip -V
pip 7.1.0 from /usr/local/lib/python2.7/dist-packages (python 2.7)
```

## Lチカ

[Raspberry Pi 2 Model B GPIO 40 Pin Block Pinout](http://www.element14.com/community/docs/DOC-73950/l/raspberry-pi-2-model-b-gpio-40-pin-block-pinout)を参考にブレッドボード配線をします。LEDのアノードは`GPIO 17`につなぎます。

* アノード (LED)                       -> GPIO17 P11 (Raspberry Pi)
* カソード (LED) -> 抵抗器1K(茶黒赤金) -> GND P14    (Raspberry Pi)

![raspi-led2.png](/2015/07/16/raspberrypi-2-headless-install-2/raspi-led2.png)


[RPi.GPIO](https://pypi.python.org/pypi/RPi.GPIO)はRaspberry Pi 2にインストール済になっています。

``` bash
$ apt-cache show python-rpi.gpio
Package: python-rpi.gpio
Source: rpi.gpio
Version: 0.5.11-1
Architecture: armhf
Maintainer: Ben Croston <ben@croston.org>
Installed-Size: 174
...
```

LチカをするPythonのプログラムを書きます。`GPIO17 P11`を使います。PINに11を指定します。


```python ~/python_apps/led-blink.py
#!/usr/bin/python
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
$ chmod +x led-blink.py
$ sudo ./led-blink.py
```


## I2Cをつかう

[I2C](http://ja.wikipedia.org/wiki/I2C)のバスを有効にします。最初に必要なパッケージをインストールします。

``` bash
$ sudo apt-get install python-smbus i2c-tools
```

raspi-configを実行してI2Cサポートを有効にします。

``` bash
$ sudo raspi-config
```

* Advanced Options > I2C

rebootします。

```bash
$ sudo reboot
```

カーネルが3.18 以上の場合は`/boot/config.txt`に以下の設定が入ります。

``` bash /boot/config.txt
dtparam=i2c_arm=on
```

`/etc/modules`にI2Cを有効にする設定します。`snd-bcm2835`の下に追記します。

```bash /etc/modules
snd-bcm2835

i2c-bcm2708
i2c-dev
```

rebootします。

```bash
$ sudo reboot
```

lsmodでi2c_devとi2c_bcm2708が表示されるとI2Cが有効になっています。

```bash
$ lsmod
Module                  Size  Used by
i2c_dev                 6047  0
snd_bcm2835            19761  0
snd_pcm                74433  1 snd_bcm2835
snd_seq                53553  0
snd_seq_device          6455  1 snd_seq
snd_timer              18205  2 snd_pcm,snd_seq
snd                    51646  5 snd_bcm2835,snd_timer,snd_pcm,snd_seq,snd_seq_device
8192cu                528381  0
i2c_bcm2708             5006  0
uio_pdrv_genirq         2958  0
uio                     8219  1 uio_pdrv_genirq
```

### BME280

Raspberry Pi 2にBME280を配線します。

```
SDI      (BME280)  -> SDA1 P02 (Raspberry Pi)
SCK      (BME280)  -> SCL1 P03 (Raspberry Pi)
GND,SDO  (BME280)  -> GND  P09 (Raspberry Pi)
Vio,CSB  (BME280)  -> 3.3v P01 (Raspberry Pi)
```

![raspi-bme280.png](/2015/07/16/raspberrypi-2-headless-install-2/raspi-bme280.png)


i2cdetectコマンドで正しく配線できたか確認します。0x76アドレスを使っています。

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
70: -- -- -- -- -- -- 76 --
```


サンプルプログラムをダウンロードします。

```bash
$ mkdir ~/python_apps
$ cd ~/python_apps
$ git clone https://github.com/IDCFChannel/bme280-meshblu-py
$ cd bme280-meshblu-py
```

bme280_sample.pyを実行すると、temp(気温)、pressure(気圧)、hum(湿度)のデータを取得することができます。

```bash
$ python bme280_sample.py
temp : 29.66  ℃
pressure : 1004.27 hPa
hum :  49.71 ％
```

## まとめ

ごちゃっとしていますが、まとめると感じになりました。

![raspi-bme280-led.png](/2015/07/16/raspberrypi-2-headless-install-2/raspi-bme280-led.png)
