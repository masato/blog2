title: "ESP8266にNodeMCUをアップロードしてHello Worldする"
date: 2015-04-05 11:04:49
categories:
 - IoT
tags:
 - ESP8266
 - NodeMCU
 - Lua
description: ESP8266にNodeMCUのファームウェアを使うと数百円でWi-Fiが使えてLuaでプログラミング可能なマイコンを作ることができます。Node.jsからハードウェアを操作できるWi-Fiモジュールを搭載したマイコンにTesselやSpark Coreなどありますがまだまだ高価です。ArduinoからWi-Fiモジュールを操作してMQTT通信をするスケッチ作成に苦労しているのでLuaから制御ができるのはとても魅力的です。
---

[ESP8266](http://www.esp8266.com/)に[NodeMCU](http://www.nodemcu.com/index_en.html)のファームウェアを使うと数百円でWi-Fiが使えてLuaでプログラミング可能なマイコンを作ることができます。Node.jsからハードウェアを操作できるWi-Fiモジュールを搭載したマイコンに[Tessel](https://tessel.io/)や[Spark Core](https://www.spark.io/)などありますがまだまだ高価です。ArduinoからWi-Fiモジュールを操作してMQTT通信をするスケッチ作成に苦労しているのでLuaから制御ができるのはとても魅力的です。


## NodeMCUのアップロード

### nodemcu-firmware

[nodemcu-firmware](https://github.com/nodemcu/nodemcu-firmware)のリポジトリから最新ファームウェアの[nodemcu_latest.bin](https://github.com/nodemcu/nodemcu-firmware/raw/master/pre_build/latest/nodemcu_latest.bin)からダウンロードします。

### nodemcu-flasher

64bit版Windows用の[nodemcu-flasher](https://github.com/nodemcu/nodemcu-flasher)を使いNodeMCUファームウェアをESP8266にアップロードします。ファームウェアは0x000000に書き込みます。Windowsホストマシンとの接続は[前回](/2015/04/05/esp8266-firmware-upload-windows/)を参考にします。ファームウェアを更新するときはESP8266のGPIO0はGNDに接続して、更新後は忘れず切断しておきます。

![nodemcu.png](/2015/04/05/esp8266-nodemcu-hello-world/nodemcu.png)

## CoolTermからHello World

[NodeMCUのExamples](http://nodemcu.com/index_en.html)を参考にCoolTermから簡単なLuaスクリプトを書いてみます。

```lua
> print("Hello World")
Hello World
> 
```

![nodemcu-helloworld.png](/2015/04/05/esp8266-nodemcu-hello-world/nodemcu-helloworld.png)


### init.lua

init.luaファイルはNodeMCUが起動したとき最初に読み込まれます。CoolTermからLuaスクリプトを書いてinit.luaを作成してみます。

最初に`file.format()`を実行してファイルの初期化をします。アップロード済みのファイルは削除されます。

```lua
> file.format()
format done.
```

print関数を記述してファイルを閉じます。

```lua
> file.open("init.lua","w")
> file.writeline([[print("Hello World!")]])
> file.close()
```

`init.lua`ファイルを読み込み文字列が書き込まれたか確認します。

```lua
> file.open("init.lua", "r")
> print(file.readline())
print("Hello World!")
> file.close()
```

## Lua Loader

CoolTermだとNodeMCUのリスタートがわかりずらいのでLua Loaderをインストールして`init.lua`の動作を確認します。一度CoolTremからESP8266を切断しておきます。

[Lua Loader](http://benlo.com/esp8266/index.html#LuaLoader)は最新版の[LuaLoader.zip 0.87](http://benlo.com/esp8266/LuaLoader.zip)のzipファイルをダウンロードして解凍します。

LuaLoader.exeを管理者権限で実行します。

メニューの`Settings > Comm Port Setting`を開きESP8266を接続したCOMポートを選択します。コンソールに`node.restart()`を入力して実行するとすぐにNodeMCUがリスタートします。`init.lua`に書いたprint関数が実行されました。

```lua
> node.restart()

NodeMCU 0.9.5 build 20150214  powered by Lua 5.1.4
Hello World!
```




