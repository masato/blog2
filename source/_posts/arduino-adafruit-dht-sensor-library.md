title: "ArduinoでDHT11デジタル温度センサーをAdafruitライブラリから使う"
date: 2015-04-19 19:58:17
categories:
 - IoT
tags:
 - Arduino
 - DHT11
 - Adafruit
 - センサー
description: ArduinoからデジタルセンサーモジュールのDHT11を使って温度と湿度を計測してみようと思います。DHTxx用のライブラリやチュートリアルはたくさん見つかるので勉強に役立ちます。DHT11よりDHT22の方が精度が高く、DHT22とAM2302は同じモジュールのようです。どれも同じライブラリを使えるようなので、安い方のDHT11を購入しました。
---

ArduinoからデジタルセンサーモジュールのDHT11を使って温度と湿度を計測してみようと思います。DHTxx用のライブラリやチュートリアルはたくさん見つかるので勉強に役立ちます。[DHT11](http://www.aitendo.com/product/10186)より[DHT22](http://akizukidenshi.com/catalog/g/gM-07002/)の方が精度が高く、DHT22とAM2302は同じモジュールのようです。どれも同じライブラリを使えるようなので、安い方のDHT11を購入しました。


<!-- more -->

## ブレッドボード配線

購入したDHT11は3ピンの完成品です。データはArduinoのデジタル側、PD2に配線します。

* `+` (DHT11) -> +5V (Arduino)
* `S` (DHT11) -> PD2 (Arduino)
* `-` (DHT11) -> GROUND (Arduino)

## adafruit/DHT-sensor-library

いつも参考にしているAdafruitのチュートリアルとライブラリを使います。

* [DHTxx Sensors](https://learn.adafruit.com/dht?view=all)
* [adafruit/DHT-sensor-library](https://github.com/adafruit/DHT-sensor-library)

リポジトリから`git clone`してライブラリをインストールします。

``` bash
$ cd ~/Documents/Arduino/libraries
$ git clone https://github.com/adafruit/DHT-sensor-library.git DHTxx
```

[DHTtester.ino](https://github.com/adafruit/DHT-sensor-library/blob/master/examples/DHTtester/DHTtester.ino)を参考にしてサンプルプログラムを用意します。

```cpp ~/Documents/Arduino/DHTxx/DHTxx.ino
#include "DHT.h"
#define DHTPIN 2
#define DHTTYPE DHT11

DHT dht(DHTPIN, DHTTYPE);

void setup() {
  Serial.begin(9600); 
  Serial.println("DHT11 test!"); 
  dht.begin();
}

void loop() {
  delay(3000);

  float h = dht.readHumidity();
  float t = dht.readTemperature();
 
  if (isnan(h) || isnan(t)) {
    Serial.println("Failed to read from DHT sensor!");
    return;
  }
  
  Serial.print("Humidity: "); 
  Serial.print(h);
  Serial.println(" %\t");
  Serial.print("Temperature: "); 
  Serial.print(t);
  Serial.println(" *C ");
}
```

コンパイルしてArduinoに書き込みます。シリアルモニタを実行すると温度と湿度が表示されます。

```
DHT11 test!
Humidity: 43.00 %	
Temperature: 25.00 *C 
Humidity: 43.00 %	
Temperature: 25.00 *C 
```

![dht11.png](/2015/04/19/arduino-adafruit-dht-sensor-library/dht11.png)