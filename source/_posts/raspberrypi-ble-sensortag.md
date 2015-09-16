title: "Raspberry Pi2からSensorTagのデータをBluebirdのCoroutineとPromiseで取得する"
date: 2015-09-03 13:11:25
categories:
 - IoT
tags:
 - RaspberryPi
 - BLE
 - SensorTag
 - ES6
 - Bluebird
 - Promise
description: BeagleBone BlackでSensorTagを使ったときのように、Raspberry Pi 2でもNode.jsのコードからデータを取得してみます。node-sensortagライブラリに付属しているtest.jsのサンプルコードはAsync.jsのasync.seriesを使っています。最近はES6のPromiseとGenerator、Coroutineで書くようにしているのでBluebirdを使うコードに書き直してみます。
---

[BeagleBone BlackでSensorTag](/2015/02/12/beagleboneblack-ble-sensortag/)を使ったときのように、Raspberry Pi 2でもNode.jsのコードからデータを取得してみます。[node-sensortag](https://github.com/sandeepmistry/node-sensortag)ライブラリに付属している[test.js](https://github.com/sandeepmistry/node-sensortag/blob/master/test.js)のサンプルコードは[Async.js](https://github.com/caolan/async)の`async.series`を使っています。最近はES6のPromiseとGenerator、Coroutineで書くようにしているので[Bluebird](https://github.com/petkaantonov/bluebird)を使うコードに書き直してみます。

<!-- more -->

## 必要なもの

### SensorTag

新しい製品の[SensorTag 2015](http://www.ti.com/ww/en/wireless_connectivity/sensortag2015/index.html)発売されていますが、以前購入した古い[SensorTag](http://www.tij.co.jp/ww/wireless_connectivity/sensortag/index.shtml)を利用します。


### Bluetooth USBアダプタ

BLE対応のアダプタはプラネックスの[BT-Micro4](https://www.planex.co.jp/products/bt-micro4/)を購入しました。Raspberry Piに挿してlsusbコマンドを実行するとデバイスが認識されています。

```bash
$ lsusb
...
Bus 001 Device 005: ID 0a12:0001 Cambridge Silicon Radio, Ltd Bluetooth Dongle (HCI mode)
```

## BlueZ

[BlueZ](http://www.bluez.org/)はLinux用のBluetooth プロトコルスタックです。[RPi Bluetooth LE](http://www.elinux.org/RPi_Bluetooth_LE)によると、Raspberry Piのapt-getからインストールできるパッケージは少し古いです。

### バージョンの確認

最初にRaspberry Pi 2のカーネルとRaspbianのバージョンを確認します。

```bash
$ uname -a
Linux raspberrypi 3.18.13-v7+ #785 SMP PREEMPT Mon May 18 17:53:02 BST 2015 armv7l GNU/Linux
$ cat /etc/debian_version
7.8
```

apt-getからインストールするbluezパッケージは4.99-2でした。

```bash
$ sudo apt-get update
$ apt-cache show bluez
...
Version: 4.99-2
```

### ソースからインストール

[RPi Bluetooth LE](http://www.elinux.org/RPi_Bluetooth_LE)に掲載されている手順でインストールしていきます。

古いパッケージがインストールされている場合は削除します。

```bash
$ sudo apt-get remove bluez
```

ビルドに必要なパッケージをインストールします。

```bash
$ sudo apt-get update
$ sudo apt-get install libdbus-1-dev libdbus-glib-1-dev libglib2.0-dev libical-dev libreadline-dev libudev-dev libusb-dev make
```

最新の5.33をビルドしてインストールします。

```bash
$ mkdir -p ~/work/bluepy
$ cd ~/work/bluepy
$ wget https://www.kernel.org/pub/linux/bluetooth/bluez-5.33.tar.gz
$ tar xzvf bluez-5.33.tar.gz
$ cd bluez-5.33
$ ./configure --disable-systemd
$ make
$ sudo make install
```

hciconfigコマンドでBLEアダプタの状態をみると`DOWN`しています。

```bash
$ hciconfig
hci0:   Type: BR/EDR  Bus: USB
        BD Address: 00:1B:DC:06:1C:CF  ACL MTU: 310:10  SCO MTU: 64:8
        DOWN
        RX bytes:55911 acl:1143 sco:0 events:2170 errors:0
        TX bytes:20152 acl:1144 sco:0 commands:370 errors:0
```

sudoでupして`UP RUNNING`の状態にします。

```bash
$ sudo hciconfig hci0 up
$ hciconfig
hci0:   Type: BR/EDR  Bus: USB
        BD Address: 00:1B:DC:06:1C:CF  ACL MTU: 310:10  SCO MTU: 64:8
        UP RUNNING
        RX bytes:56475 acl:1143 sco:0 events:2199 errors:0
        TX bytes:20270 acl:1144 sco:0 commands:399 errors:0
```


## Node.jsのインストール

Node.jsで[Bluebird](https://github.com/petkaantonov/bluebird)を使う場合、`--harmony`オプションと`0.11+`が必要になります。[node_arm](http://node-arm.herokuapp.com/)から最新の`0.12.6`をRaspberry Piにインストールします。

```bash
$ wget http://node-arm.herokuapp.com/node_latest_armhf.deb
$ sudo dpkg -i node_latest_armhf.deb
$ npm -v
2.11.2
$ node -v
v0.12.6
```

## プロジェクト

[nose-sensortag](https://github.com/sandeepmistry/node-sensortag)を使ったNode.jsのサンプルコードを書いていきます。リポジトリは[こちら](https://github.com/masato/node-bluebird-sensortag)です。プロジェクトのディレクトリを作成して移動します。

```bash
$ mkdir -p ~/node_apps/bluebird-sensortag
$ cd !$
```

[gitignore.io](https://www.gitignore.io)から`.gitignore`をダウンロードします。

```bash
$ curl https://www.gitignore.io/api/linux,node > .gitignore
```

### package.json

package.jsonに必要なパッケージを`dependencies`フィールドに定義します。node-sensortagに付属しているサンプル用にAsync.jsも使います。`scripts`フィールドには`--harmony`フラグを追加します。

```json ~/node_apps/bluebird-sensortag/package.json
{
    "name": "node-bluebird-sensortag",
    "description": "node-bluebird-sensortag",
    "version": "0.0.1",
    "private": true,
    "dependencies": {
        "sensortag": "~1.1.1",
        "async": "~1.4.2",
        "bluebird": "~2.9.34"
    },
    "scripts": {
        "start": "node --harmony app.js"
    }
}
```

パッケージをインストールします。

```bash
$ npm install
```

### app.js

[Promise.coroutine](https://github.com/petkaantonov/bluebird/blob/master/API.md#promisecoroutinegeneratorfunction-generatorfunction---function)で使うGenerator内部では、yieldする関数はPromiseを返す必要があります。node-sensortagのライブラリはPromiseを返さないため[Promisification](https://github.com/petkaantonov/bluebird/blob/master/API.md#promisification)が必要になります。

今回は[Promise.promisifyAll](https://github.com/petkaantonov/bluebird/blob/master/API.md#promisepromisifyallobject-target--object-options---object)を使い、`SensorTag`オブジェクトの関数をすべてPromisificationします。Promisificationされた関数には`Async`がサフィックスされます。

[test.js](https://github.com/sandeepmistry/node-sensortag/blob/master/test.js)にあるように本来は`sensorTag.enableIrTemperature(callback);`として、callbackに取得したデータを渡します。Promiseにすると`then`で取得することができます。非同期のコードがとても見やすくなりました。

```js ~/node_apps/bluebird-sensortag/app.js
'use strict';

var Promise = require('bluebird'),
    SensorTag = require('sensortag');

Promise.promisifyAll(SensorTag);

SensorTag.discover(function(sensorTag) {
    console.log('discovered: ' + sensorTag);

    sensorTag.on('disconnect', function() {
        console.log('Tag Disconnected');
        process.exit(0);
    });

    Promise.coroutine(function* () {
        console.log('connectAndSetUp');
        yield sensorTag.connectAndSetUpAsync();

        console.log('enableIrTemperature');
        yield sensorTag.enableIrTemperatureAsync();

        yield Promise.delay(2000);

        console.log('readIrTemperature');
        yield sensorTag.readIrTemperatureAsync()
                       .then(function(data) {
                           console.log('\tobject temperature = %d °C',
                                       data[0].toFixed(1));
                           console.log('\tambient temperature = %d °C',
                                       data[1].toFixed(1));
                       });

        console.log('disableIrTemperature');
        yield sensorTag.disableIrTemperatureAsync();

        console.log('enableHumidity');
        yield sensorTag.enableHumidityAsync();

        yield Promise.delay(2000);

        console.log('readHumidity');
        yield sensorTag.readHumidityAsync()
                       .then(function(data) {
            console.log('\ttemperature = %d °C', data[0].toFixed(1));
            console.log('\thumidity = %d %', data[1].toFixed(1));
        });

        console.log('enableMagnetometer');
        yield sensorTag.enableMagnetometerAsync();

        console.log('enableBarometricPressure');
        yield sensorTag.enableBarometricPressureAsync();

        yield Promise.delay(2000);

        console.log('readBarometricPressure');
        yield sensorTag.readBarometricPressureAsync()
                       .then(function(data) {
            console.log('\tpressure = %d hPa', data.toFixed(1));
        });

        console.log('disableBarometricPressure');
        sensorTag.disableBarometricPressureAsync();

        console.log('disconnect');
        yield sensorTag.disconnectAsync();
    })();
});
```

## プログラムの実行

### 付属のtest.js

SensorTagの電源を入れてから、node-sensortag付属の[test.js](https://github.com/sandeepmistry/node-sensortag/blob/master/test.js)をsudoで実行してみます。

```bash
$ cd ~/node_apps/node-bluebird-sensortag/node_modules/sensortag
$ sudo node test.js
discovered: {"id":"b4994c34d5c0","type":"cc2540"}
connectAndSetUp
readDeviceName
        device name = TI BLE Sensor Tag
readSystemId
        system id = b4:99:4c:00:00:34:d5:c0
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
        object temperature = 24.6 °C
        ambient temperature = 27.4 °C
disableIrTemperature
enableAccelerometer
readAccelerometer
        x = 0 G
        y = 0.2 G
        z = 3.9 G
disableAccelerometer
enableHumidity
readHumidity
        temperature = 27.7 °C
        humidity = 66.4 %
disableHumidity
enableMagnetometer
readMagnetometer
        x = -58.5 μT
        y = 89.3 μT
        z = 7.5 μT
disableMagnetometer
enableBarometricPressure
readBarometricPressure
        pressure = 1002.9 mBar
disableBarometricPressure
enableGyroscope
readGyroscope
        x = 2.3 °/s
        y = 2 °/s
        z = 0.3 °/s
disableGyroscope
readTestData
        data = 63
readTestConfiguration
        configuration = 0
readSimpleRead - waiting for button press ...
left: false
right: true
disconnect
disconnected!
```

SensorTagからBLEでRaspberry Piにデータを送信して、Node.jsのプログラムから取得することができました。

### app.jsの実行

取得している環境データは減らしていますが、BluebirdのCoroutineで書いたコードも正常に動作しました。

```bash
$ sudo npm start

> node-bluebird-sensortag@0.0.1 start /home/pi/node_apps/node-bluebird-sensortag
> node --harmony app.js

discovered: {"id":"b4994c34d5c0","type":"cc2540"}
connectAndSetUp
enableIrTemperature
readIrTemperature
        object temperature = 23.6 °C
        ambient temperature = 27.2 °C
disableIrTemperature
enableHumidity
readHumidity
        temperature = 27.6 °C
        humidity = 61.4 %
enableMagnetometer
enableBarometricPressure
readBarometricPressure
        pressure = 1005.8 hPa
disableBarometricPressure
disconnect
Tag Disconnected
```