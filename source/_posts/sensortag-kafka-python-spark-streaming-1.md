title: "PythonとSensorTag, Kafka, Spark Streamingのストリーム処理 - Part 1: Raspberry Pi 3"
date: 2017-07-27 13:15:19
categories:
 - IoT
tags:
 - Spark
 - Kafka
 - Python
 - SensorTag
description: Raspberry Pi 3からSensorTagの環境データを取得します。
---

　このシリーズではRaspberry Pi 3からSensorTagの環境データを取得します。その後にKafkaを経由したSpark Streamingでウィンド分析するPythonのコードを書いてみます。
　
　Raspberry Pi 3のセットアップ方法は多くの[サンプル](https://masato.github.io/2017/01/29/eclipse-iot-kura-install/)があります。ここでは簡単にSensorTagの環境データを取得するための準備をします。

<!-- more -->

## 用意するもの

　Raspberry Pi 3はWi-FiとBluetoothを内蔵しています。ドングルが不要になり用意するものが減りました。この例ではUSB-A端子がある古いMacBook Pro (macOS Sierra 10.12.6)を開発用端末に使っています。Raspberry Pi 3とEthernet接続する周辺機器を用意します。
　
* [USB2.0 有線LANアダプタ](https://www.amazon.co.jp/dp/B00LVH885U)
* [Ethernetケーブル](https://www.amazon.co.jp/dp/B008RVY7K8/)


　テキサスインスツルメンツのSensorTagはRaspberry Pi 3とBluetooth接続して温度や湿度といった環境データを簡単に取得することができます。
　
* [CC2650](http://www.tij.co.jp/tool/jp/TIDC-CC2650STK-SENSORTAG)

## macOSでRaspbian Liteイメージを焼く

　オフィシャルの[Installing operating system images on Mac OS](https://www.raspberrypi.org/documentation/installation/installing-images/mac.md)の手順に従います。SDカードのデバイス名を確認してunmountします。この例では/dev/disk2です。

```
$ diskutil list
...
/dev/disk2 (internal, physical):
   #:                       TYPE NAME                    SIZE       IDENTIFIER
   0:     FDisk_partition_scheme                        *15.6 GB    disk2
   1:             Windows_FAT_32 NO NAME                 15.6 GB    disk2s1

$ diskutil unmountDisk /dev/disk2
```

　Raspbian Jessie Liteを[ダウンロード](https://www.raspberrypi.org/downloads/raspbian/)してSDカードに焼きます。SSHの有効化も忘れずに行います。

```
$ cd ~/Downloads
$ wget http://director.downloads.raspberrypi.org/raspbian_lite/images/raspbian_lite-2017-07-05/2017-07-05-raspbian-jessie-lite.zip
$ unzip 2017-07-05-raspbian-jessie-lite.zip
$ sudo dd bs=1m if=2017-07-05-raspbian-jessie-lite.img of=/dev/rdisk2
$ touch /Volumes/boot/ssh
$ diskutil unmountDisk /dev/disk2
```

### mDNSでSSH接続する

　macOSはRaspberry Pi 3とEthernetケーブル接続をして簡単にSSHできます。デフォルトでは以下のユーザーが設定されています。

* username: pi
* password: raspberry

```
$ ssh pi@raspberrypi.local
```

　macOSの公開鍵をRaspberry Pi 3の`~/.ssh/authorized_keys'にコピーします。

```
$ mkdir -p -m 700 ~/.ssh
$ cat << EOF > ~/.ssh/authorized_keys
ssh-rsa AAAA...
EOF
$ chmod 600 ~/.ssh/authorized_keys
```

　ログインし直します。

```
$ exit
```

### 無線LAN

　無線LANのアクセスポイント(ESSID)を設定してネットワークを再起動します。`ping`からインターネット接続を確認します。

```
$ sudo sh -c 'wpa_passphrase [ESSID] [パスワード] >> /etc/wpa_supplicant/wpa_supplicant.conf'
$ sudo ifdown wlan0
$ sudo ifup wlan0
$ ping -c 1 www.yahoo.co.jp
```

### raspi-config

　`raspi-config`でパスワードとタイムゾーンを変更します。

```
$ sudo raspi-config
```

* 1 Change User Password
* 4 Localisation Options
 * I2 Change Timezone
 * Asia > Tokyo


### apt-get

　日本のミラーサイトに変更してパッケージを最新にします。ファイルを編集するためにvimをインストールします。

```
$ sudo sed -i".bak" -e "s/mirrordirector.raspbian.org/ftp.jaist.ac.jp/g" /etc/apt/sources.list
$ sudo apt-get update && sudo apt-get dist-upgrade -y
$ sudo apt-get install vim -y
```

## BluetoothとSensorTag


　SensorTagの電源を入れてRaspberry Pi 3からスキャンします。BDアドレスを確認します。
 
```
$ sudo hcitool lescan
...
B0:B4:48:BE:5E:00 CC2650 SensorTag
...
```

### gatttool


　`gatttool`コマンドのインタラクティブモードで確認したBDアドレスのSensorTagに接続します。

```
$ gatttool -b B0:B4:48:BE:5E:00 --interactive
[B0:B4:48:BE:5E:00][LE]> connect
Attempting to connect to B0:B4:48:BE:5E:00
Connection successful
[B0:B4:48:BE:5E:00][LE]>
```

　[Raspberry PiでTI製センサータグ(CC2650)の値を取得してみる](http://blog.livedoor.jp/sce_info3-craft/archives/8932936.html)を参考にしてセンサーの値を取得します。

 * 0x24(config)に01を書いて有効にする
 * 0x21からバイトを取得する

```
[B0:B4:48:BE:5E:00][LE]> char-write-cmd 0x24 01
[B0:B4:48:BE:5E:00][LE]> char-read-hnd 0x21
Characteristic value/descriptor: 94 0b f0 0d
```

　4バイトの値が得られます。[SensorTag User Guide](http://processors.wiki.ti.com/index.php/SensorTag_User_Guide#IR_Temperature_Sensor)によると順番に以下のデータになります。

```
ObjLSB ObjMSB AmbLSB AmbMSB
```

　Python REPLから取得した4バイトを摂氏に変換して周辺温度と物体温度を出力します。

```python
>>> raw_data = '94 0b f0 0d'
>>> rval = raw_data.split()
>>> obj_temp = int(rval[1] + rval[0], 16) / 4
>>> amb_temp = int(rval[3] + rval[2], 16) / 4
>>> obj_temp_celcius = obj_temp * 0.03125
>>> amb_temp_celcius = amb_temp * 0.03125
>>> print("周囲温度 : {:.2f} ℃".format(amb_temp_celcius))
周囲温度 : 27.88 ℃
>>> print("物体温度 : {:.2f} ℃".format(obj_temp_celcius))
物体温度 : 23.16 ℃
```

### bluepyのsensortagコマンド

　PythonからBluetooth LEのデバイスを操作するために[bluepy](https://github.com/IanHarvey/bluepy.git)をインストールします。

```
$ sudo apt-get install python-pip libglib2.0-dev -y
$ sudo pip install bluepy
```

　`sensortag`コマンドが使えるようになります。`-T`フラグを付けBDアドレスを引数に実行します。

```
$ sensortag -T B0:B4:48:BE:5E:00
```

　`-T`フラグを指定すると周囲温度、物体温度の順番に取得できます。

```
Connecting to B0:B4:48:BE:5E:00
('Temp: ', (28.34375, 24.25))
('Temp: ', (28.375, 23.28125))
...
```

### SensorTag

　周囲温度(ambient)と物体温度(object)、湿度(humidy)を取得するPythonのスクリプトを書きます。

```python ~/python_apps/temp.py 
# -*- coding: utf-8 -*-
from bluepy.sensortag import SensorTag
import sys
import time

def main():
    argvs = sys.argv
    argc = len(argvs)
    if (argc != 2):
        print 'Usage: # python {0} bd_address'.format(argvs[0])
        quit()
    host = argvs[1]
    print('Connecting to ' + host)

    timeout = 5.0
    count = 3

    tag = SensorTag(host)

    tag.IRtemperature.enable()
    tag.humidity.enable()

    time.sleep(1.0)

    counter = 0
    while True:
        counter += 1
        tAmb, tObj = tag.IRtemperature.read()

        print("温度: 周囲: {0:.2f}, 物体: {1:.2f}".format(tAmb, tObj) )
        humidy, RH = tag.humidity.read()
        print("湿度: humidy: {0:.2f}, RH: {1:.2f}".format(humidy, RH))

        if counter >= count:
           break

        tag.waitForNotifications(timeout)

    tag.disconnect()
    del tag

if __name__ == '__main__':
    main()
```


　SensorTagのBDアドレスを引数にスクリプトを実行します。


```
$ python temp.py B0:B4:48:BE:5E:00
```

　5秒間隔で3回計測した結果を出力します。エアコンが効いてきたので快適になりました。

```
Connecting to B0:B4:48:BE:5E:00
温度: 周囲: 25.03, 物体: 19.94
湿度: humidy: 25.25, RH: 69.47
温度: 周囲: 25.06, 物体: 20.69
湿度: humidy: 25.25, RH: 69.37
温度: 周囲: 25.06, 物体: 20.69
湿度: humidy: 25.25, RH: 69.37
```