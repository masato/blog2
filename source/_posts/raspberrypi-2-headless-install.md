title: "Raspberry Pi 2とセンサーデータで遊ぶための初期設定"
date: 2015-05-21 13:40:43
categories:
 - IoT
tags:
 - RaspberryPi
 - RaspberryPi2
 - Lチカ
 - DHT11
 - DS18B20
 - センサー
description: Raspberry Pi Model Bが壊れてしまったので新しくRaspberry Pi 2を購入しました。T-Cobblerを間違えて配線してしまったら電源が入らなくなりました。最近不安定だったのでそろそろ買い換え時期だったかも知れません。Model Bで作業していた環境を最初から設定していきます。
---

Raspberry Pi Model Bが壊れてしまったので新しくRaspberry Pi 2を購入しました。T-Cobblerを間違えて配線してしまったら電源が入らなくなりました。最近不安定だったのでそろそろ買い換え時期だったかも知れません。Model Bで作業していた環境を最初から設定していきます。

<!-- more -->

## USB-TTLシリアル変換ケーブル

[PL2303HX内蔵USB-Serial変換ケーブル](http://www.amazon.co.jp/gp/product/B00L8SP7U6)を使ってOSXとRaspberry Pi 2を接続します。[Software Installation (Mac)]( https://learn.adafruit.com/adafruits-raspberry-pi-lesson-5-using-a-console-cable?view=all)を参考にしてGPIOピンにつなぎます。

* VCC 赤 (USB-TTL) -> 5V  P2  (Raspberry Pi)
* GND 黒 (USB-TTL) -> GND P6  (Raspberry Pi)
* RXD 白 (USB-TTL) -> TXD GPIO14 P8  (Raspberry Pi)
* TXD 緑 (USB-TTL) -> RXD GPIO15 P10 (Raspberry Pi)

OSXからScreenでターミナル接続します。

``` bash
$ screen /dev/cu.usbserial 115200
```

## 無線LAN

無線LANのUSBドングルは[BUFFALO WLI-UC-GNM](http://www.amazon.co.jp/dp/B003NSAMW2)を使います。SSIDステルスモードの環境なので`scan_ssid=1`が必要です。

``` bash
$ sudo sh -c 'wpa_passphrase {ssid} {passphrase} >> /etc/wpa_supplicant/wpa_supplicant.conf'
$ sudo vi /etc/wpa_supplicant/wpa_supplicant.conf
ctrl_interface=DIR=/var/run/wpa_supplicant GROUP=netdev
update_config=1
network={
        scan_ssid=1
        ssid="xxx"
        proto=WPA2
        key_mgmt=WPA-PSK
        psk=xxxx
}
```

`/etc/network/interfaces`を編集します。ローミングは不要なので`wpa-roam`の箇所は`wpa-conf`にする必要があります。

``` bash /etc/network/interfaces
auto lo

iface lo inet loopback
iface eth0 inet dhcp

auto wlan0
allow-hotplug wlan0
iface wlan0 inet dhcp
wpa-conf /etc/wpa_supplicant/wpa_supplicant.conf
iface default inet dhcp
```

wlan0を再起動します。

``` bash
$ sudo ifdown wlan0
$ sudo ifup wlan0
```

IPアドレスの確認をして、pingで名前解決もできるようになりました。

```
$ ip addr show wlan0
$ ping -c 1 www.yahoo.co.jp
PING www.g.yahoo.co.jp (183.79.197.250) 56(84) bytes of data.
64 bytes from 183.79.197.250: icmp_req=1 ttl=51 time=15.7 ms

--- www.g.yahoo.co.jp ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 15.718/15.718/15.718/0.000 ms
```


## raspi-configの初期設定

raspi-configをroot権限で実行して、Raspberry Pi 2の初期設定をしていきます。

``` bash
$ sudo raspi-config
```

以下の箇所だけ最低限変更しておきます。

* Expand Filesystem > Select
* Internationalisation Options > Change Locale > ja_JP.UTF-8 > Default
* Internationalisation Options > Change Timezone > Asia > Tokyo

再起動します。

``` bash
$ sudo reboot
```

再起動後にファイルシステムがリサイズされました。ようやくパッケージをインストールする準備ができました。

``` bash
$ df -h
ファイルシス   サイズ  使用  残り 使用% マウント位置
rootfs            15G  2.4G   12G   18% /
/dev/root         15G  2.4G   12G   18% /
devtmpfs         460M     0  460M    0% /dev
tmpfs             93M  228K   93M    1% /run
tmpfs            5.0M     0  5.0M    0% /run/lock
tmpfs            186M     0  186M    0% /run/shm
/dev/mmcblk0p1    56M   15M   42M   26% /boot
```


## ファームウェアの更新

最初にカーネルの初期状態を確認します。

```
$ cat /proc/version
Linux version 3.18.7-v7+ (dc4@dc4-XPS13-9333) (gcc version 4.8.3 20140303 (prerelease) (crosstool-NG linaro-1.13.1+bzr2650 - Linaro GCC 2014.03) ) #755 SMP PREEMPT Thu Feb 12 17:20:48 GMT 2015
```

パッケージの更新とアップグレードをします。

``` bash
$ sudo apt-get update
$ sudo apt-get upgrade
```

apt-getではカーネルは変わらないのでファームウェアの更新をします。

``` bash
$ sudo rpi-update
...
 *** A reboot is needed to activate the new firmware
```

再起動します。

``` bash
$ sudo reboot
```

ファームウェアを更新してカーネルのバージョンが`3.18.13-v7+`にあがりました。

``` bash
$ cat /proc/version
Linux version 3.18.13-v7+ (dc4@dc4-XPS13-9333) (gcc version 4.8.3 20140303 (prerelease) (crosstool-NG linaro-1.13.1+bzr2650 - Linaro GCC 2014.03) ) #785 SMP PREEMPT Mon May 18 17:53:02 BST 2015
```
## その他設定

### vim

エディタはvimを使います。

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
```


## Python

Raspberry Pi 2でセンサーデータを取得する場合はPythonを使うことが多いです。Pythonの開発環境をインストールします。

``` bash
$ sudo apt-get update
$ sudo apt-get install python-dev
$ curl https://bootstrap.pypa.io/get-pip.py -o - | sudo python
$ pip -V
pip 6.1.1 from /usr/local/lib/python2.7/dist-packages (python 2.7)
```

## I2Cをつかう

[I2C](http://ja.wikipedia.org/wiki/I2C)のバスを有効にします。最初に必要なパッケージをインストールします。

``` bash
$ sudo apt-get install python-smbus i2c-tools
```

raspi-configを実行してI2Cサポートを有効にします。

* Advanced Options > I2C

再起動します。

``` bash
$ sudo reboot
```

piユーザーでもi2cデバイスを操作できるようにi2cグループを追加します。

``` bash
$ sudo adduser pi i2c
```

カーネルが3.18 以上の場合は`/boot/config.txt`に以下の設定が入ります。

``` bash /boot/config.txt
dtparam=i2c_arm=on
```

`/etc/modules`にI2Cを有効にする設定を追記します。

```bash /etc/modules
snd-bcm2835

i2c-bcm2708 
i2c-dev
```

lsmodでi2c_devとi2c_bcm2708が表示されるとI2Cが有効になっています。

``` bash
$ lsmod
Module                  Size  Used by
ctr                     3709  2
ccm                     7771  2
i2c_dev                 6027  0
snd_bcm2835            18365  0
snd_pcm                73475  1 snd_bcm2835
snd_seq                53078  0
snd_seq_device          5628  1 snd_seq
snd_timer              17784  2 snd_pcm,snd_seq
snd                    51038  5 snd_bcm2835,snd_timer,snd_pcm,snd_seq,snd_seq_device
...
```


### DHT11で温度を計測する

I2Cの温度湿度センサーモジュールは[DHT11](http://www.aitendo.com/product/10186)の完成品を使います。Pythonのパッケージは[Adafruit_Python_DHT](https://github.com/adafruit/Adafruit_Python_DHT)をインストールします。Adafruit-Raspberry-Pi-Python-Codeの[Adafruit_DHT_Driver_Python]( https://github.com/adafruit/Adafruit-Raspberry-Pi-PythonCode/blob/master/Adafruit_DHT_Driver_Python/DEPRECATED.txt)はDEPRECATEDになっています。

``` bash
$ mkdir -p ~/python_apps
$ cd $_
$ git clone https://github.com/adafruit/Adafruit_Python_DHT.git
$ cd Adafruit_Python_DHT
$ sudo python setup.py install
```

ブレッドボード配線をします。DHT11は完成品なので3pinになっています。

* DATA (DHT11) -> GPIO4 P7 (Raspberry Pi)
* VCC  (DHT11) -> 3.3v  P1 (Raspberry Pi)
* GND  (DHT11) -> GND   P9 (Raspberry Pi)


Adafruit_Python_DHTのサンプルプログラムを実行して温度を計測します。

``` bash
$ cd ~/python_apps/Adafruit_Python_DHT/examples
$ sudo ./AdafruitDHT.py 11 4
Temp=28.0*C  Humidity=34.0%
```

## 1-Wireをつかう

[1-Wire](http://ja.wikipedia.org/wiki/1-Wire)のバスを有効にするために`/boot/config.txt`に以下を追記します。

``` bash /boot/config.txt
dtparam=i2c_arm=on
dtoverlay=w1-gpio-pullup,gpiopin=4
```

再起動します。

``` bash
$ sudo reboot
```

再起動後にw1-gpioとw1-thermのカーネルモジュールをロードします。

``` bash
$ sudo modprobe w1-gpio
$ sudo modprobe w1-therm
```

### DS18B20で温度を計測する

1-Wire用のセンサーは完成品の[1820-3PL](http://www.aitendo.com/product/10812)を使います。

カーネルモジュールをロードすると/sys/bus/w1/devicesディレクトリが作成されます。

``` bash
$ cd /sys/bus/w1/devices
$ ls
00-800000000000  w1_bus_master1
```

ブレッドボード配線をします。完成品なので3pinになっています。
 
* `+` (DS18B20) -> 3.3V  P1 (Raspberry Pi)
* `S` (DS18B20) -> GPIO4 P7 (Raspberry Pi)
* `-` (DS18B20) -> GND   P9 (Raspberry Pi)

配線後に`28-`で始まる値は温度センサーのデバイスIDのディレクトリができます。

``` bash
$ cd /sys/bus/w1/devices
$ ls
00-200000000000  28-0314626a2cff  w1_bus_master1
```

`28-`のディレクトリに移動してw1_slaveを読み込み温度を計測します。

``` bash
$ cd 28-0314626a2cff
$ cat w1_slave
c4 01 55 00 7f ff 0c 10 06 : crc=06 YES
c4 01 55 00 7f ff 0c 10 06 t=28250
$ cat w1_slave
c5 01 55 00 7f ff 0c 10 45 : crc=45 YES
c5 01 55 00 7f ff 0c 10 45 t=28312
```

## GPIOでLチカする

[WiringPi](http://wiringpi.com/)と[RPi.GPIO](https://pypi.python.org/pypi/RPi.GPIO)のパッケージをそれぞれインストールしてGPIOを使い定番のLチカをしてみます。

### WiringPi

GPIOを使うためC言語のモジュールの[WiringPi](http://wiringpi.com/)をビルドします。

``` bash
$ cd
$ git clone git://git.drogon.net/wiringPi
$ cd wiringPi
$ ./build
```

gpioコマンドを実行してGPIOの番号確認します。ピンアサインはよく忘れるので一覧で表示されて便利です。

``` bash
$ gpio readall
+-----+-----+---------+------+---+---Pi 2---+---+------+---------+-----+-----+
 | BCM | wPi |   Name  | Mode | V | Physical | V | Mode | Name    | wPi | BCM |
 +-----+-----+---------+------+---+----++----+---+------+---------+-----+-----+
 |     |     |    3.3v |      |   |  1 || 2  |   |      | 5v      |     |     |
 |   2 |   8 |   SDA.1 | ALT0 | 1 |  3 || 4  |   |      | 5V      |     |     |
 |   3 |   9 |   SCL.1 | ALT0 | 1 |  5 || 6  |   |      | 0v      |     |     |
 |   4 |   7 | GPIO. 7 |   IN | 1 |  7 || 8  | 0 | ALT0 | TxD     | 15  | 14  |
 |     |     |      0v |      |   |  9 || 10 | 1 | ALT0 | RxD     | 16  | 15  |
 |  17 |   0 | GPIO. 0 |   IN | 0 | 11 || 12 | 0 | IN   | GPIO. 1 | 1   | 18  |
 |  27 |   2 | GPIO. 2 |   IN | 0 | 13 || 14 |   |      | 0v      |     |     |
 |  22 |   3 | GPIO. 3 |   IN | 0 | 15 || 16 | 0 | IN   | GPIO. 4 | 4   | 23  |
 |     |     |    3.3v |      |   | 17 || 18 | 0 | IN   | GPIO. 5 | 5   | 24  |
 |  10 |  12 |    MOSI |   IN | 0 | 19 || 20 |   |      | 0v      |     |     |
 |   9 |  13 |    MISO |   IN | 0 | 21 || 22 | 0 | IN   | GPIO. 6 | 6   | 25  |
 |  11 |  14 |    SCLK |   IN | 0 | 23 || 24 | 1 | IN   | CE0     | 10  | 8   |
 |     |     |      0v |      |   | 25 || 26 | 1 | IN   | CE1     | 11  | 7   |
 |   0 |  30 |   SDA.0 |   IN | 1 | 27 || 28 | 1 | IN   | SCL.0   | 31  | 1   |
 |   5 |  21 | GPIO.21 |  OUT | 1 | 29 || 30 |   |      | 0v      |     |     |
 |   6 |  22 | GPIO.22 |   IN | 1 | 31 || 32 | 0 | IN   | GPIO.26 | 26  | 12  |
 |  13 |  23 | GPIO.23 |   IN | 0 | 33 || 34 |   |      | 0v      |     |     |
 |  19 |  24 | GPIO.24 |   IN | 0 | 35 || 36 | 0 | IN   | GPIO.27 | 27  | 16  |
 |  26 |  25 | GPIO.25 |   IN | 0 | 37 || 38 | 0 | IN   | GPIO.28 | 28  | 20  |
 |     |     |      0v |      |   | 39 || 40 | 0 | IN   | GPIO.29 | 29  | 21  |
 +-----+-----+---------+------+---+----++----+---+------+---------+-----+-----+
 | BCM | wPi |   Name  | Mode | V | Physical | V | Mode | Name    | wPi | BCM |
 +-----+-----+---------+------+---+---Pi 2---+---+------+---------+-----+-----+
```

ブレッドボード配線をします。LEDのアノードは`GPIO 17`につなぎます。
 
* アノード (LED)                   -> GPIO17 P11 (Raspberry Pi)
* カソード (LED) -> 抵抗器1K(茶黒赤) -> GND P14    (Raspberry Pi)

GPIO-17を出力にします。

``` bash
$ gpio -g mode 17 out
```

LEDを点灯します。

``` bash
$ gpio -g write 17 1
```

LEDを消灯します。

``` bash
$ gpio -g write 17 0
```

### RPi.GPIO

[RPi.GPIO](https://pypi.python.org/pypi/RPi.GPIO)のdebパッケージのpython-rpi.gpioはRaspberry Pi 2にインストール済みでした。

``` bash
$ sudo apt-get install python-rpi.gpio
パッケージリストを読み込んでいます... 完了
依存関係ツリーを作成しています
状態情報を読み取っています... 完了
python-rpi.gpio はすでに最新バージョンです。
アップグレード: 0 個、新規インストール: 0 個、削除: 0 個、保留: 4 個。
```

LチカをするPythonのプログラムを書きます。ブレッドボード配線はWiringPiのときと同じで`GPIO 17`を使います。

```python ~/python_apps/led_blink.py
#!/usr/bin/env python
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

GPIOの制御はroot権限で実行します。LEDを3回点滅してプログラムは終了します。

``` bash
$ chmod u+x led_blink.py
$ sudo ./led_blink.py
```
