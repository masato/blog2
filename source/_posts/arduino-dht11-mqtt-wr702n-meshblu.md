title: "ArduinoからDHT11の温度と湿度データをMQTTを使ってMeshbluに送信する"
date: 2015-04-20 13:33:04
categories:
 - IoT
tags:
 - Arduino
 - DHT11
 - Adafruit
 - MQTT
 - Meshblu
 - センサー
description: ArduinoのEthernetライブラリを使ってモバイルルーター経由でHTTPクライアントの実装を確認しました。またDHT11のデジタル温度センサーからAdafruitのライブラリを使い温度と湿度を計測できるようになりました。ようやく準備が整ったのでMeshbluブローカーにMQTT通信でセンシングデータを送信してみます。
---

ArduinoのEthernetライブラリを使ってモバイルルーター経由で[HTTPクライアント](/2015/04/18/ardiono-restclient-ethernet-wifi/)の実装を確認しました。またDHT11のデジタル温度センサーからAdafruitのライブラリを使い[温度と湿度を計測](/2015/04/19/arduino-adafruit-dht-sensor-library/)できるようになりました。ようやく準備が整ったのでMeshbluブローカーにMQTT通信でセンシングデータを送信してみます。

<!-- more -->

## 参考

ArduinoのMQTTライブラリはNick O'Leary氏の[Arduino Client for MQTT](http://knolleary.net/arduino-client-for-mqtt/)を使います。またArduinoからMQTTを使うコードは以下の記事を参考にしました。

* [Arduino Uno と IBM IoT Foundation を利用してクラウド対応の温度センサーを作成する: 第 2 回 スケッチを作成して IBM IoT Foundation Quickstart に接続する](http://www.ibm.com/developerworks/jp/cloud/library/cl-bluemix-arduino-iot2/index.html)
* [Using MQTT to connect Arduino to the Internet of Things](http://chrislarson.me/blog/using-mqtt-connect-arduino-internet-things.html)
* [Publishing Arduino Sensor Data through MQTT over Ethernet](http://e.verything.co/post/61576413925/publishing-arduino-sensor-data-through-mqtt-over)


## Mesubluサーバーでデバイスの登録

Mesubluのコンテナを起動しているDockerホストにログインして作業をします。デバイスのトークンはPythonでランダムな文字列を作成します。

``` python
>>> import random
>>> num = 8
>>> arr = list('0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ')
>>> print("".join([random.choice(arr) for i in range(num)]))
FwsFjZy4
```

ArduinoからDHT11のセンシングデータを送受信するためのデバイスを登録します。

``` bash
$ curl -X POST \
  localhost \
  -d "name=freeboard&uuid=freeboard&token=FwsFjZy4"
  {"geo":null,"ipAddress":"172.17.42.1","name":"freeboard","online":false,"timestamp":"2015-04-20T02:51:31.841Z","uuid":"freeboard","token":"FwsFjZy4"}
```

Mosquittoのクライアントを使いArduinoからpublishされたデータを受信できるようにします。

``` bash
$ mosquitto_sub \
  -h localhost \
  -p 1883 \
  -t freeboard \
  -u freeboard \
  -P FwsFjZy4 \
  -d
```

## Arduino IDE

Arduino IDEのホストマシンはOSXを使います。

### ライブラリのダウンロード

Arduino IDEのlibrariesディレクトリにライブラリをインストールします。
DHT11のライブラリはAdafruitの[DHT-sensor-library](https://github.com/adafruit/DHT-sensor-library)リポジトリから`git clone`します。

``` bash
$ cd ~/Documents/Arduino/libraries
$ git clone https://github.com/adafruit/DHT-sensor-library.git DHTxx
```

MQTTクライアントはリポジトリの[アーカイブページ](https://github.com/knolleary/pubsubclient/archive)から最新版をダウンロードして解凍します。

``` bash
$ cd ~/Documents/Arduino/libraries
$ wget https://github.com/knolleary/pubsubclient/archive/v1.9.1.tar.gz
$ tar zxvf v1.9.1.tar.gz
$ rm v1.9.1.tar.gz
```

メニューから、開く > libraries > pubsubclient-1.9.1 > PubSubClientにサンプルを選べるようになります。

* mqtt_auth
* mqtt_basic
* mqtt_publish_in_callback

mqtt_authを例にしてスケッチを書いていきます。

### スケッチの作成

Arduino IDEを使いスケッチを作成してArduino Unoに書き込みます。`buildJson()`関数でMQTT通信するメッセージをJSON形式で作成しています。MesubluブローカーにMQTT通信でメッセージを送信する場合、トピック名は`message`が固定になります。`devices`キーに送信先デバイスのUUID(freeboard)を指定して、`payload`キーにメッセージ本文を人にのJSON形式で記述します。

実際に送信するデータは文字列です。JSONを文字列にエスケープすると以下のようになります。

```cpp
"{\"devices\": \"freeboard\", \"payload\": {\"humidity\":42,\"temperature\":26}}"
```

```cpp ~/Documents/Arduino/mqtt_auth/mqtt_auth.ino
#include <SPI.h>
#include <Ethernet.h>
#include <PubSubClient.h>
#include <DHT.h>

#define DHTPIN 2
#define DHTTYPE DHT11

#define MQTT_SERVER     "{MESUBLU_BROKER}"
#define DEVICE_UUID     "freeboard"
#define DEVICE_PASSWORD "FwsFjZy4"
#define MAC_ADDRESS_STR  "90A2DA89C850"

DHT dht(DHTPIN, DHTTYPE);

byte MAC_ADDRESS[] = { 0x90, 0xA2, 0xDA, 0x89, 0xC8, 0x50 };
float humidity = 0.0;
float tempC = 0.0;

EthernetClient ethClient;
PubSubClient client(MQTT_SERVER, 1883, callback, ethClient);

void setup()
{
  Serial.begin(9600); 
  Serial.println("DHT11 and MQTT test");
  
  dht.begin();
  Ethernet.begin(MAC_ADDRESS);
}

void getData() {
  humidity = dht.readHumidity();
  tempC = dht.readTemperature();
}

String buildJson() {
  String json = "{";
    json +=  "\"devices\": \"freeboard\"";
    json += ",";
    json += "\"payload\":";
    json += "{";
    json += "\"humidity\":";
    json += humidity;
    json += ",";
    json += "\"temperature\":";
    json += tempC;
    json += "}";
    json += "}";
    
  return json;
}

void loop()
{  
  char clientStr[36];
  sprintf(clientStr,"%s/%s",DEVICE_UUID,MAC_ADDRESS_STR);
  
  getData();
  
  if (isnan(humidity) || isnan(tempC)) {
    Serial.println("Failed to read from DHT sensor!");
    return;
  }
  
  if (!client.connected()) {
    client.connect(clientStr,DEVICE_UUID,DEVICE_PASSWORD);
  }
  
  if (client.connected() ) {
    String json = buildJson();
    char jsonStr[200];
    json.toCharArray(jsonStr,200);
    
    boolean pubresult = client.publish("message",jsonStr);
    //boolean pubresult = client.publish("message","{\"devices\": \"freeboard\", \"payload\": {\"humidity\":42,\"temperature\":26}}");
    if (pubresult)
      Serial.println("successfully sent");
    else
      Serial.println("unsuccessfully sent");
  } else {
    Serial.println("not connected");
  }
  delay(5000);
}

// Handles messages arrived on subscribed topic(s)
void callback(char* topic, byte* payload, unsigned int length) {
}
```

シリアルモニタを開くと5秒間隔でDHT11から温度と湿度を計測して、MeshbluのMQTTブローカーにpublishします。

```
DHT11 and MQTT test
successfully sent
successfully sent
successfully sent
```

![dht11-mqtt.png](/2015/04/20/arduino-dht11-mqtt-wr702n-meshblu/dht11-mqtt.png)

## Mesubluサーバーでsubscribeの確認

さきほど`mosquitto_sub`コマンドでsubscribeしたシェルをみるとデータの受信が確認できます。
  
``` bash
...
{"topic":"message","data":{"devices":"freeboard","payload":{"humidity":40,"temperature":26},"fromUuid":"freeboard"}}
Client mosqsub/2903-minion1.cs received PUBLISH (d0, q0, r0, m0, 'freeboard', ... (116 bytes))
{"topic":"message","data":{"devices":"freeboard","payload":{"humidity":40,"temperature":26},"fromUuid":"freeboard"}}
Client mosqsub/2903-minion1.cs received PUBLISH (d0, q0, r0, m0, 'freeboard', ... (116 bytes))
```

