title: "ESP8266のファームウェアをWindowsからアップロードする"
date: 2015-04-04 20:44:35
categories:
 - IoT
tags:
 - ESP8266
 - NodeMCU
 - Lua
 - PL2303HX
description: NodeMCUのnodemcu-flasherを使ってESP8266にWindowsからファームウェアをアップロードしてみます。NodeMCUのファームウェアはLuaスクリプトでマイクロコントローラを操作できるようです。ATコマンドで動作確認したいので前回と同様にElectricdragonのAT-0942ファームウェアで確認します。
---

[NodeMCU](http://www.nodemcu.com/index_en.html)の[nodemcu-flasher](https://github.com/nodemcu/nodemcu-flasher)を使ってESP8266にWindowsからファームウェアをアップロードしてみます。NodeMCUのファームウェアはLuaスクリプトでマイクロコントローラを操作できるようです。ATコマンドで動作確認したいので前回と同様に[Electricdragon](http://www.electrodragon.com/w/ESP8266_Firmware)のAT-0942ファームウェアで確認します。

<!-- more -->

## ブレッドボード配線


PL2303HXを内蔵した[USB-TTLシリアル変換ケーブル](http://www.amazon.co.jp/dp/B00L8SP7U6)ロジックレベルは3.3V、電源は5Vです。ESP8266は3.3V動作なのでArduino Unoの3.3Vから電源を供給します。ESP8266のファームウェアを更新するときはGPIO0をGNDに接続します。

![ESP8266-firm.png](/2015/04/04/esp8266-firmware-upload-windows/ESP8266-firm.png)

## ファームウェア

[Electricdragon](http://www.electrodragon.com/w/ESP8266_Firmware)のファームウェアは[Oldフォルダ](https://drive.google.com/folderview?id=0B_ctPy0pJuW6d1FqM1lvSkJmNU0&usp=sharing)からAT-0942.binを使います。

## nodemcu-flasher

[nodemcu-flasher](https://github.com/nodemcu/nodemcu-flasher)から64bit版の実行ファイルをダウンロードして実行します。ダウンロードしたファームウェアを0x000000に書き込みます。

![firmware.png](/2015/04/04/esp8266-firmware-upload-windows/firmware.png)

CoolTermからATコマンドを実行してファームウェアの更新を確認します。

![coolterm.png](/2015/04/04/esp8266-firmware-upload-windows/coolterm.png)



