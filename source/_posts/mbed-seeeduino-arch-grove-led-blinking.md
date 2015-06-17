title: "mbedのSeeeduino ArchとGroveのLEDを使ってLチカする"
date: 2015-03-11 23:55:08
tags:
 - mbed
 - SeeeduinoArch
 - Lチカ
 - Grove
 - ARM
description: Seeeduino Archと一緒に購入したGrove Blue LEDを使ってLチカしてみます。Grove Starter Kit for ArduinoやGrovePi+ Starter Kit for Raspberry Piと違い、Wikiにmbedのサンプルコードがないのでちょっと困りました。

---

Seeeduino Archと一緒に購入した[Grove Blue LED](http://www.seeedstudio.com/depot/Grove-Blue-LED-p-1139.html)を使ってLチカしてみます。[Grove Starter Kit for Arduino](http://www.seeedstudio.com/depot/Grove-Starter-Kit-for-Arduino-p-1855.html)や[GrovePi+ Starter Kit for Raspberry Pi](http://www.seeedstudio.com/depot/GrovePi-Starter-Kit-for-Raspberry-Pi-p-2240.html)と違い、[Wiki](http://www.seeedstudio.com/wiki/Grove_-_LED)にmbedのサンプルコードがないのでちょっと困りました。

<!-- more -->


## Grove Blue LED

[GROVE - 青 LED (5mm)](https://www.switch-science.com/catalog/1251/)をスイッチサイエンスから購入しました。4ピンケーブルは付属してました。別途購入してしまいましたが。


![grobe-blue-led.jpg](/2015/03/11/mbed-seeeduino-arch-grove-led-blinking/grobe-blue-led.jpg)


## ピンアウト図

[Arch　V1.1](http://developer.mbed.org/users/seeed/notebook/Arch_V1_1/)のピンアウト図を見るとGroveのポートは3つあります。

![arch_v1.1_pinout.png](/2015/03/11/mbed-seeeduino-arch-grove-led-blinking/arch_v1.1_pinout.png)

マイコンボードによってピン番号が異なり、Arduinoの`digital port 2`やRaspberry Piの`D4`といった表記になっていません。Seeeduino Archの場合は以下のピン番号を使うとLチカするようです。

* UARTポート: `P1_14`
* I2Cポート: `P0_04`

## mbed Compiler

オンラインの[mbed Compiler](https://developer.mbed.org)で[前回](/2015/03/10/mbed-seeeduino-arch-setting-up/)コンパイルした`mbed_blinky`プロジェクトと同様に`Blinky LED Hello World`テンプレートを使います。ピン番号を`LED1`から`P1_14`に変更しただけです。

### main.cpp

```cpp main.cpp
#include "mbed.h"

DigitalOut myled(P1_14);

int main() {
    while(1) {
        myled = 1;
        wait(0.2);
        myled = 0;
        wait(0.2);
    }
}
```

### ファームウェアの書き込み

OSXにUSBケーブルで接続して、Seeeduino Archのリセットボタンを押します。D0が青色に点灯し、OSXのデスクトップに`CRP DISABLD`ボリュームがマウントされたことを確認します。

![crp-disabled.png](/2015/03/11/mbed-seeeduino-arch-grove-led-blinking/crp-disabled.png)

mbed Compilerからコンパイルを実行するとバイナリファイルがダウンロードできます。ddコマンドを使ってSeeeduino Archに書き込みます。

``` bash
$ dd if=~/Downloads/grove_led_LPC11U24.bin of=/Volumes/CRP\ DISABLD/firmware.bin conv=notrunc
20+1 records in
20+1 records out
10308 bytes transferred in 0.000077 secs (133854135 bytes/sec)
```

Seeeduino Archのリセットボタンを押すと「ディスクの不正な取り出し」の警告がでますがLチカが始まります。

![alerts.png](/2015/03/11/mbed-seeeduino-arch-grove-led-blinking/alerts.png)
