title: "BeagleBone BlackにBLEアダプタを挿してSensorTagからセンサーデータを取得する"
date: 2015-02-12 00:10:09
tags:
 - BeagleBoneBlack
 - EclipseOrion
 - BLE
 - SensorTag
description: BeagleBone BlackのSDカードにあるUbuntu14.04.1を使います。BLE対応のUSBアダプタを挿してSensorTagからセンサーデータを取得してみます。簡単なNode.jsのプログラムは、前回インストールしたEclipse Orion上で開発します。
---


BeagleBone BlackのSDカードにあるUbuntu14.04.1を使います。BLE対応のUSBアダプタを挿してSensorTagからセンサーデータを取得してみます。簡単なNode.jsのプログラムは、[前回インストール](/2015/02/10/beagleboneblack-eclipse-orion/)したEclipse Orion上で開発します。

<!-- more -->

## Ubuntu 14.04.1

Ubuntuのリリースを確認します。

``` bash
$ cat /etc/lsb-release
DISTRIB_ID=Ubuntu
DISTRIB_RELEASE=14.04
DISTRIB_CODENAME=trusty
DISTRIB_DESCRIPTION="Ubuntu 14.04.1 LTS"
```

カーネル情報です。

``` bash
$ uname -a
Linux arm 3.14.31-ti-r49 #1 SMP PREEMPT Sat Jan 31 14:17:42 UTC 2015 armv7l armv7l armv7l GNU/Linux
```

## SensorTag

[SensorTag](http://www.tij.co.jp/ww/wireless_connectivity/sensortag/index.shtml)には温度センサーや湿度センターなど複数のセンサー搭載されています。コイン電池をセットしてキットを組み立てるだけです。

## Bluetooth LE対応USBアダプタ

SensorTagはBeagleBone Blackに挿したBluetoothLEのUSBアダプタの[BT-Micro4](http://www.amazon.co.jp/dp/B0071TE1G2)と通信をします。lsusbコマンドで認識されているか確認します。

``` bash
$ lsusb -v

Bus 001 Device 002: ID 0a12:0001 Cambridge Silicon Radio, Ltd Bluetooth Dongle (HCI mode)
Couldn't open device, some information will be missing
Device Descriptor:
  bLength                18
  bDescriptorType         1
  bcdUSB               2.00
  bDeviceClass          224 Wireless
  bDeviceSubClass         1 Radio Frequency
  bDeviceProtocol         1 Bluetooth
  bMaxPacketSize0        64
  idVendor           0x0a12 Cambridge Silicon Radio, Ltd
  idProduct          0x0001 Bluetooth Dongle (HCI mode)
  bcdDevice           88.91
  iManufacturer           0
  iProduct                2
  iSerial                 0
  bNumConfigurations      1
  Configuration Descriptor:
```

### Node.jsのsensortagモジュール

Node.jsのSensorTag用のモジュールは[sensortag](https://www.npmjs.com/package/sensortag)を使います。はじめに依存パッケージをインストールします。

``` bash
$ sudo apt-get update
$ sudo apt-get install bluez libbluetooth-dev
```

sensortagをインストールすると[noble](https://github.com/sandeepmistry/noble)も一緒に入ります。

``` bash
$ npm install sensortag
noble install: platform is "linux'
noble install: running node-gyp ...
make: Entering directory `/home/ubuntu/node_modules/sensortag/node_modules/noble/build'
  CC(target) Release/obj.target/hci-ble/src/hci-ble.o
  LINK(target) Release/hci-ble
  LINK(target) Release/hci-ble: Finished
  CC(target) Release/obj.target/l2cap-ble/src/l2cap-ble.o
  LINK(target) Release/l2cap-ble
  LINK(target) Release/l2cap-ble: Finished
make: Leaving directory `/home/ubuntu/node_modules/sensortag/node_modules/noble/build'
noble install: done
sensortag@0.1.9 node_modules/sensortag
└── noble@0.3.8 (debug@0.7.4)
```

hciconfigコマンドを使いデバイスが認識されているか確認します。

``` bash
$ hciconfig
hci0:   Type: BR/EDR  Bus: USB
        BD Address: 00:1B:DC:06:1C:CF  ACL MTU: 310:10  SCO MTU: 64:8
        UP RUNNING PSCAN
        RX bytes:1158 acl:0 sco:0 events:63 errors:0
        TX bytes:1050 acl:0 sco:0 commands:63 errors:0
```

### sensortag付属のテストスクリプト

sensortagモジュールに同梱されているテストスクリプトを使います。SensorTagの電源を入れてからsudoで実行します。

``` bash
$ cd ~/node_modules/sensortag
$ sudo node test.js
connect
discoverServicesAndCharacteristics
readDeviceName
        device name = TI BLE Sensor Tag
readSystemId
        system id = b4:99:4c:0:0:34:34:c0
readSerialNumber
        serial number = N.A.
readFirmwareRevision
        firmware revision = 1.4 (Jul 12 2013)
readHardwareRevision
        hardware revision = N.A.
readSoftwareRevision
        software revision = N.A.
readManufacturerName
        manufacturer name = Texas Instruments
enableIrTemperature
readIrTemperature
        object temperature = 20.6 °C
        ambient temperature = 24.3 °C
disableAccelerometer
enableAccelerometer
readAccelerometer
        x = 0.2 G
        y = -1.2 G
        z = 0.8 G
disableAccelerometer
enableHumidity
readHumidity
        temperature = 24.6 °C
        humidity = 42.1 %
disableHumidity
enableMagnetometer
readMagnetometer
        x = -57.5 μT
        y = 102.9 μT
        z = -4.3 μT
disableMagnetometer
enableBarometricPressure
readBarometricPressure
        pressure = -93 mBar
disableBarometricPressure
enableGyroscope
readGyroscope
        x = -250 °/s
        y = 139 °/s
        z = -21.9 °/s
disableGyroscope
readSimpleRead
```

## Orionで簡単なサンプル作成

[Orion](http://eclipse.org/orion/)のWebIDEを起動して簡単なサンプルを作ります。[iot-beaglebone](https://github.com/ibm-messaging/iot-beaglebone)を参考にします。

最初にデフォルトのnanoから、vimにエディタを変更します。

``` bash
$ sudo apt-get install vim
$ sudo update-alternatives --set editor /usr/bin/vim.basic
```


### npmのパスを通す

Orionからnpmを実行できるようにパスを通します。npmの場所を確認します。

``` bash
$ which npm
/usr/bin/npm
$ sudo find / -type f -name "npm-cli.js"
/usr/share/npm/bin/npm-cli.js
$ which npm
/usr/bin/npm
$ ls -al /usr/bin/npm
lrwxrwxrwx 1 root root 27 Oct 22  2013 /usr/bin/npm -> ../share/npm/bin/npm-cli.js
```

orion.confのnpm_pathにnpmの場所を指定します。

```bash ~/node_modules/orion/orion.conf
## Path to npm-cli.js on your computer
#npm_path=/path/to/npm-cli.js
npm_path=/usr/bin/npm
```

Orionを再起動します。

``` bash
$ pm2 restart orion
[PM2] restartProcessId process id 0
┌──────────┬────┬──────┬──────┬────────┬───────────┬────────┬────────────┬──────────┐
│ App name │ id │ mode │ PID  │ status │ restarted │ uptime │     memory │ watching │
├──────────┼────┼──────┼──────┼────────┼───────────┼────────┼────────────┼──────────┤
│ orion    │ 0  │ fork │ 1379 │ online │         1 │ 0s     │ 4.316 MB   │ disabled │
└──────────┴────┴──────┴──────┴────────┴───────────┴────────┴────────────┴──────────┘
 Use `pm2 info <id|name>` to get more details about an app
```

### プログラムの作成

最初にプロジェクトのフォルダを作成します。

* File > Folder > sensor-tag

package.jsonに必要なモジュールを定義します。

```json package.json
{
  "name": "sensortag-test",
  "version": "0.0.1",
  "private": true,
  "dependencies": {
    "async": "*.*.*",
    "sensortag": "*.*.*"
  },
  "scripts": {"start": "node app.js"}
}
```

メインプログラムです。

```js app.js
/*eslint-env node */

var SensorTag = require('sensortag');
var async = require('async');

SensorTag.discover(function(sensorTag) {
  sensorTag.on('disconnect', function() {
    console.log('Tag Disconnected');
    process.exit(0);
  });

  async.series([
    function(callback) {
      console.log('SensorTag connected');
      sensorTag.connect(callback);
    },
    function(callback) {
      console.log('disconnect');
      sensorTag.disconnect(callback);
    }
  ]);
});
```

テスト用に簡単なプロジェクトを作成しました。

![nsensor-tag-project.png](/2015/02/12/beagleboneblack-ble-sensortag/sensor-tag-project.png)


Shell画面に移動してpackage.jsonからモジュールをインストールします。

``` bash
$ npm install
...
noble install: done
async@0.9.0 node_modules/async

sensortag@0.1.9 node_modules/sensortag
└── noble@0.3.8 (debug@0.7.4)
```

### ファイルにケーパビリティを割り当てる

setcapコマンドを使い適切な権限を設定して、sudoを使わなくてもプログラムを実行できるようにします。

``` bash
$ sudo apt-get install libcap2-bin
$ cd /home/ubuntu/node_modules/orion/.workspace/sensortag-test
$ find -path '*noble*Release/hci-ble' -exec sudo setcap cap_net_raw+eip '{}' \;
```

Shell画面に移動して、SensorTagの電源を入れてからプログラムを実行します。


![npm-start.png](/2015/02/12/beagleboneblack-ble-sensortag/npm-start.png)
