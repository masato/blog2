title: "Arduinoのespduinoを使いESP8266からMQTT通信する"
date: 2015-04-04 13:12:42
tags:
 - Arduino
 - espduino
 - ESP8266
 - MQTT
categories:
 - IoT
---

ESP8266のWi-Fiモジュールと[espduino](https://github.com/tuanpmt/espduino)を使うとArduinoのシリアルポートからSLIPプロトコルでインターネット接続ができるようになります。espduinoはESP8266のファームウェアです。このファームウェアではMQTTとRESTをサポートしているので、Arduinoをクラウドに接続するために便利に使えそうです。


## OSXのPython環境

ESP8266のファームウェア更新はPythonの[esptool](https://github.com/themadinventor/esptool/)を使います。OSXには[pyenv](https://github.com/yyuu/pyenv)を使いPython仮想環境をインストールします。

### pyenv

Homebrewからpyenvをインストールします。

``` bash
$ brew update
$ brew install pyenv
```

`~/.bash_profile`にpyenvのロードを設定を書いてシェルを再起動します。

``` bash
$ echo 'eval "$(pyenv init -)"' >> ~/.bash_profile
```

pyenvでインストール可能なPythonのバージョンを一覧します。

``` bash
$ pyenv install -l
```

Python 2.7.9をインストールします。最初はzlibが見つからずビルドエラーになりました。

``` bash
$ pyenv install 2.7.9
...
Installing Python-2.7.9...
ERROR: The Python zlib extension was not compiled. Missing the zlib?

Please consult to the Wiki page to fix the problem.
https://github.com/yyuu/pyenv/wiki/Common-build-problems


BUILD FAILED (OS X 10.10.2 using python-build 20141028)
```

Wikiを読むと[Build failed: "ERROR: The Python zlib extension was not compiled. Missing the zlib?"](https://github.com/yyuu/pyenv/wiki/Common-build-problems#build-failed-error-the-python-zlib-extension-was-not-compiled-missing-the-zlib)のページに`CFLAGS`を追加するように指示が書いてあります。

``` bash
$ CFLAGS="-I$(xcrun --show-sdk-path)/usr/include" pyenv install 2.7.9
Downloading Python-2.7.9.tgz...
-> https://www.python.org/ftp/python/2.7.9/Python-2.7.9.tgz
Installing Python-2.7.9...
Installing setuptools from https://bootstrap.pypa.io/ez_setup.py...
Installing pip from https://bootstrap.pypa.io/get-pip.py...
Installed Python-2.7.9 to /Users/mshimizu/.pyenv/versions/2.7.9
```

インストール済みのPythonを確認します。まだシステムのPythonを使っている状態です。

``` bash
$ pyenv versions
* system (set by /Users/mshimizu/.pyenv/version)
  2.7.9
```

現在使われているPythonのバージョンを確認しても同様に`system`が選択されています。

``` bash
$ pyenv version
system (set by /Users/mshimizu/.pyenv/version)
```

今回は特定ディレクトリでpyenvのPythonを使いたいので`pytnv local`を実行します。
 
``` bash
$ cd ~/arduino_apps/espduino
$ pyenv local  2.7.9
```


``` bash
$ which python
/Users/mshimizu/.pyenv/shims/python
$ python -V
Python 2.7.9
$ pip -V
pip 6.0.8 from /Users/mshimizu/.pyenv/versions/2.7.9/lib/python2.7/site-packages (python 2.7)
```


### pySerial

 * pySerial

``` bash
$ pip install pyserial
...
Successfully installed pyserial-2.7
```




## espduinoファームウェアのアップロード

esptoolはespduinoに同梱されているバージョンを使います。

ESP8266のGPIO0をArduinoのGNDに接続してファームウェアのアップロードをします。

 * OSXで作業

``` bash
$ cd ~/arduino_apps
$ git clone https://github.com/tuanpmt/espduino
$ cd espduino
```


[Getting started with the esp8266 and Arduino](http://www.madebymarket.com/blog/dev/getting-started-with-esp8266.html)

``` bash
$ ./esptool.py -p /dev/tty.usbserial-A7045L3R write_flash 0x000000 esp8266.9.2.2.bin
```



### アップロード


## ブレッドボード配線

前回はSoftwareSerialを使いArduinoのPIN2(RX)、PIN3(TX)とESP8266を接続しましたが、今回はPIN2(RX)、PIN3(TX)はデバッグ用にUSB-TTL変換ケーブルと接続します。ArduinoのPIN0(RX0)、PIN1(TX0)の5VとESP8266の3.3Vはロジックレベル変換モジュールを介して接続します。

* RX (ESP8266 3.3V) -> logic level converter -> TX0 (Arduino 5V)
* TX (ESP8266 3.3V) -> logic level converter -> RX0 (Arduino 5V)
* CH_PD (ESP8266) -> VCC (Arduino 3.3V)
* VCC (ESP8266) -> VCC (Arduino 3.3V)
* GND (ESP8266) -> GND (Arduino)
* RX (USB-TTL) -> PIN3 (Arduino)
* TX (USB-TTL) -> PIN2 (Arduino)
* GND (USB-TTL) -> GND (Arduino)

## テスト

