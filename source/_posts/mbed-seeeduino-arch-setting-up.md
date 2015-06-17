title: "mbedのSeeeduino ArchをセットアップしてLチカする"
date: 2015-03-10 22:49:30
tags:
 - mbed
 - SeeeduinoArch
 - Lチカ
 - OSX
 - ARM
description: 最近ARMとIBMが発表したmbed IoT Starter KitのFRDM-K64Fや、マイクロソフトのGR版 IoT Kit（仮称）のGR-SAKURAなど、IoTプラットフォームにmbedのマイコンボードを採用する例が出てきました。Arduinoと同じ位置づけになります。Arduinoに比べると拡張基板が少なかったり、割高なハードウェアが多いのですが、Seeeduino Archが比較的安く購入できたのでまずはLチカまでやってみます。
---

最近ARMとIBMが発表した[mbed IoT Starter Kit](https://developer.ibm.com/iot/recipes/arm-mbed-starterkit-ethernet/)の[FRDM-K64F](http://developer.mbed.org/platforms/FRDM-K64F/)や、マイクロソフトの[GR版 IoT Kit（仮称）](http://ms-iotkithol-jp.github.io/Prepare.htm)の[GR-SAKURA](http://www.core.co.jp/product/m2m/gr-peach/)など、IoTプラットフォームにmbedのマイコンボードを採用する例が出てきました。Arduinoと同じ位置づけになります。Arduinoに比べると拡張基板が少なかったり、割高なハードウェアが多いのですが、[Seeeduino Arch](http://www.seeedstudio.com/wiki/Seeeduino_Arch)が比較的安く購入できたのでまずはLチカまでやってみます。

<!-- more -->

セットアップ手順は基本的に[mbedを始めましょう！](http://developer.mbed.org/users/nxpfan/notebook/lets_get_started_jp/)の通りですがいくつか情報が古くなっているようです。

## ホストマシンと接続

mbedの開発はホストマシンとUSBケーブルで接続して利用します。今回はホストマシンにOSXを利用します。Seeeduino Archを箱からだして、OSXと接続するとmicroUSBコネクター横のLEDが以下のように光ります。ARM mbed Developer Siteの[Seeeduino Arch](http://developer.mbed.org/users/seeed/notebook/Arch_V1_1/)ページにピン配置図があります。

* LED_USB: 青で点灯
* LED1-4: 赤 > 緑 > 黄 > 青と移動しながら点滅

![arch_v1.1_pinout.png](/2015/03/10/mbed-seeeduino-arch-setting-up/arch_v1.1_pinout.png)

OSXには「ARCH」という名前でボリュームがマウントされます。

![seeeduino-arch-device.png](/2015/03/10/mbed-seeeduino-arch-setting-up/seeeduino-arch-device.png)

`ARCH`ボリュームにある`ARCH.HTM`ファイルを実行すると[Arch V1.1](http://developer.mbed.org/users/seeed/notebook/Arch_V1_1/)のページが開きます。
 
## ARM mbed Developer Site

### サインアップ

先ほど表示したページの右上にサインアップボタンがあるので必要な情報を入れ、[ARM mbed Developer Site](http://developer.mbed.org/)に[ユーザー登録](https://developer.mbed.org/account/login/?next=/)をします。

### mbed Compilerにプラットフォームの追加

ログインしたらメニューのプラットフォームから、今回接続しているSeeeduino Archの画像をクリックします。

* Platforms > Seeeduino Arch

![platforms-seeeduino-arch.png](/2015/03/10/mbed-seeeduino-arch-setting-up/platforms-seeeduino-arch.png)

[Seeeduino Arch](https://developer.mbed.org/platforms/Seeeduino-Arch/)のページからmbed CompilerにSeeeduino Archをプラットフォームとして追加します。mbedはオンラインIDEを使ってプログラミングとコンパイルができるが特徴の一つです。

* Followボタンを押してフォローする
* Add to your mbed Compilerボタンを押して、オンラインIDEに追加する

![seeeduino-arch-page.png](/2015/03/10/mbed-seeeduino-arch-setting-up/seeeduino-arch-page.png)

## Lチカ

### オンラインIDEでコンパイル

mbed Compilerが起動すると`mbed_blinky`プロジェクトを作成するダイアログが表示されます。OKボタンを押してプロジェクトを作成します。

![mbed-compiler.png](/2015/03/10/mbed-seeeduino-arch-setting-up/mbed-compiler.png)

プロジェクトにはメインプログラムが用意されています。

```cpp main.cpp
#include "mbed.h"

DigitalOut myled(LED1);

int main() {
    while(1) {
        myled = 1;
        wait(0.2);
        myled = 0;
        wait(0.2);
    }
}
```

Compile ボタンを実行すると`mbed_blinky_LPC11U24.bin`のバイナリファイルが作成されるので、OSXにダウンロードします。


### FinderでARCHボリュームにコピーできない。

ダウンロードしたバイナリファイルを`ARCH`ボリュームにFinderからコピーしようとすると領域不足でコピーできないというメッセージが表示されて失敗します。

![copy-fail.png](/2015/03/10/mbed-seeeduino-arch-setting-up/copy-fail.png)

[Programming Seeeduino Arch(LPC11U24) on Windows, Linux or Mac](http://developer.mbed.org/users/seeed/notebook/programming-seeeduino-arch/)によると、OSXやLinuxの場合はddコマンドを使ってファームウェアの書き込みをする必要があります。

### USB-ISPモードで起動

Seeeduino Arch (LPC11U24)は[USB-ISP (In-System-Programming)](http://www.nxp-lpc.com/development.html)を使ってファームウェアの更新をします。ArchをOSXにUSBケーブルで接続して、DCジャック下にあるリセットボタンを長押しするとUSB-ISPモードで起動します。ボリューム名が「CRP DISABLD」となってマウントされます。

![crp-disabled.png](/2015/03/10/mbed-seeeduino-arch-setting-up/crp-disabled.png)

### ddコマンドでファームウェア書き込み

Linuxの場合はディスクを{mnt_dir}にマウントしてからddコマンドを使います。

``` bash
$ dd if={new_firmware.bin} of={mnt_dir}/firmware.bin conv=notrunc
```

OSXの場合は以下の書式になります。

``` bash
$ dd if={new_firmare.bin} of=/Volumes/CRP\ DISABLD/firmware.bin conv=notrunc
```

mbed Compilerからダウンロードしたバイナリファイルをddコマンドを使って書き込みます。

``` bash
$ dd if=~/Downloads/mbed_blinky_LPC11U24.bin of=/Volumes/CRP\ DISABLD/firmware.bin conv=notrunc
20+1 records in
20+1 records out
10308 bytes transferred in 0.000083 secs (124238177 bytes/sec)
```

すぐに書き込みは終了するので`CRP DISABLD`ボリュームをアンマウントします。

``` bash
$ sudo umount /Volumes/CRP\ DISABLD
```

### Lチカの実行

DCジャック下のリセットボタンをちょっと押して、ファームウェアを更新するとLED1が赤く点滅を始めます。もう一度ボタンを押すと`CRP DISABLD`ボリュームがUSB-ISPモードでマウントされLED_USBが青く点灯します。
