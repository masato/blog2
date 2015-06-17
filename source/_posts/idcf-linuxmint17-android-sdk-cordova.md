title: "Linux Mint 17.1 MATEにAndroid SDKとCordovaをインストールする"
date: 2015-01-01 09:18:11
tags:
 - IDCFクラウド
 - LinuxMint17
 - MATE
 - DockerDevEnv
 - Ionic
 - Cordova
 - AndroidSDK
description: Docker上に構築したAndroid開発環境はCLI環境が前提になります。画面の確認はIonicのWebサーバーに接続してWebブラウザ経由で行いました。Androidのエミュレーターを起動するためにはデスクトップ環境が必要になります。Linux Mint 17をインストールした確認した手順通りに、新しくLinux Mint 17.1をインストールしてAndroid開発環境を構築していきます。
---


Docker上に[構築した](/2014/12/30/docker-devenv-ionic-cordova/)Android開発環境はCLI環境が前提になります。画面の確認はIonicのWebサーバーに接続してWebブラウザ経由で行いました。Androidのエミュレーターを起動するためにはデスクトップ環境が必要になります。[Linux Mint 17をインストール](/2014/12/31/idcf-linuxmint17-xrdp/)した確認した手順通りに、新しくLinux Mint 17.1をインストールしてAndroid開発環境を構築していきます。

<!-- more -->

### Linux Mint 17.1のインストール

17.1の仮想マシンをIDCFクラウド上に用意しました。

``` bash
$ cat /etc/lsb-release
DISTRIB_ID=LinuxMint
DISTRIB_RELEASE=17.1
DISTRIB_CODENAME=rebecca
DISTRIB_DESCRIPTION="Linux Mint 17.1 Rebecca"
```

### 依存パッケージのインストール

Androidの開発環境を構築するため、依存しているパッケージをインストールしていきます。Gitをインストールします。

``` bash
$ sudo apt-get update
$ sudo apt-get install git
```

Oracle JDK 7と、CordovaとIonicで必要になるAntをインストールします。

``` bash
$ sudo add-apt-repository ppa:webupd8team/java
$ sudo sh -c 'echo oracle-java7-installer shared/accepted-oracle-license-v1-1 select true | /usr/bin/debconf-set-selections'
$ sudo apt-get update && sudo apt-get install oracle-java7-installer oracle-java7-set-default ant
```

一度ログアウトして`JAVA_HOME`など、Javaの開発環境を確認します。

``` bash
$ exit
$ echo $JAVA_HOME
/usr/lib/jvm/java-7-oracle
$ java -version
java version "1.7.0_72"
Java(TM) SE Runtime Environment (build 1.7.0_72-b14)
Java HotSpot(TM) 64-Bit Server VM (build 24.72-b04, mixed mode)
$ ant -version
Apache Ant(TM) version 1.9.3 compiled on April 8 2014
```

Android SDKは一部で32bit版のライブラリが必要になります。[How to install ia32-libs in Ubuntu 14.04 LTS (Trusty Tahr)](http://stackoverflow.com/questions/23182765/how-to-install-ia32-libs-in-ubuntu-14-04-lts-trusty-tahr)を参考にして必要なパッケージをインストールします。

``` bash
$ sudo apt-get update
$ sudo apt-get install lib32z1 lib32ncurses5 lib32bz2-1.0 lib32stdc++6
```

### Android SDKのインストール

xrdpでLinux Mint 17.1のデスクトップにログインしてから作業を続けます。Android SDKを[ダウンロード](http://developer.android.com/sdk/index.html)して解凍します。

``` bash
$ wget http://dl.google.com/android/android-sdk_r24.0.2-linux.tgz
$ tar zxvf android-sdk_r24.0.2-linux.tgz
```

Android SDKを解凍したディレクトリのPATHを追加します。

``` bash
$ echo 'export ANDROID_HOME=${HOME}/android-sdk-linux' >> ${HOME}/.bashrc
$ echo 'export PATH=${PATH}:${ANDROID_HOME}/tools:${ANDROID_HOME}/platform-tools' >> ${HOME}/.bashrc
$ source ~/.bashrc
```

`android`コマンドを実行してAndroid SDK Managerが起動します。

``` bash
$ android
```

デフォルトの選択状態にAndroid 4.4.2 (API 19)を追加します。

* Tools -> Android SDK Tools 24.0.2
* Tools -> Android SDK Platform-tools 21
* Tools -> Android SDK Build-tools 21.1.2
* Android 5.0.1 (API 21) 
* Android 4.4.2 (API 19)
* Extras -> Android Support Library 21.03

`Install`ボタンをクリック後、ライセンスに同意してインストールします。

{% img center /2015/01/01/idcf-linuxmint17-android-sdk-cordova/xrdp-android.png %}

adb (Android Debug Bridge) インストールを確認します。

``` bash
$ adb version
Android Debug Bridge version 1.0.32
```

### Node.jsをnvmでインストールする

Node.jsをnvmを使ってインストールします。

``` bash
$ sudo apt-get install curl
$ curl https://raw.githubusercontent.com/creationix/nvm/v0.22.0/install.sh | bash
$ source ${HOME}/.nvm/nvm.sh
$ nvm install v0.10
$ nvm use v0.10
Now using node v0.10.35
$ nvm alias default v0.10
default -> v0.10 (-> v0.10.35)
```

`~/.bashrc`に環境変数の設定が入りました。

``` bash ~/.bashrc
export ANDROID_HOME=${HOME}/android-sdk-linux
export PATH=${PATH}:${ANDROID_HOME}/tools:${ANDROID_HOME}/platform-tools


export NVM_DIR="/home/mshimizu/.nvm"
[ -s "$NVM_DIR/nvm.sh" ] && . "$NVM_DIR/nvm.sh"  # This loads nvm
```


### Cordovaのインストール

Cordovaはnvmでインストールしたnpmを使ってインストールします。

``` bash
$ npm install -g cordova
```

Cordovaのバージョンを確認します。

``` bash
$ cordova --version
4.1.2
```

### AVD (Android Virtual Device)のインストール

Androidの開発を行うため、AVD (Android Virtual Device)をインストールします。まず利用できるターゲットを確認します。今回はid: 3のAPI level 19、armeabi-v7aのAVDを利用します。

``` bash
$ android list targets
...
----------
id: 3 or "Google Inc.:Google APIs:19"
     Name: Google APIs
     Type: Add-On
     Vendor: Google Inc.
     Revision: 10
     Description: Android + Google APIs
     Based on Android 4.4.2 (API level 19)
     Libraries:
      * com.google.android.media.effects (effects.jar)
          Collection of video effects
      * com.android.future.usb.accessory (usb.jar)
          API for USB Accessories
      * com.google.android.maps (maps.jar)
          API for Google Maps
     Skins: HVGA, QVGA, WQVGA400, WQVGA432, WSVGA, WVGA800 (default), WVGA854, WXGA720, WXGA800, WXGA800-7in
 Tag/ABIs : default/armeabi-v7a
...
```

`--target`フラグに確認したid: 3を指定してAVDを作成します。

``` bash
$ android create avd --name test --target 3
Auto-selecting single ABI armeabi-v7a
Created AVD 'test' based on Google APIs (Google Inc.), ARM (armeabi-v7a) processor,
with the following hardware config:
hw.lcd.density=240
hw.ramSize=512
vm.heapSize=48
```

### Cordovaのサンプルアプリの作成

cordovaコマンドを使いサンプルアプリを作成します。

``` bash
$ cordova create cordova_test com.example.test "CordovaTestApp"
$ cd cordova_test 
$ cordova platform add android
```

buildが成功したら、エミュレーターを起動します。

``` bash
$ cordova build
$ cordova emulate
```

Cordovaのエミュレーターが表示されました。

{% img center /2015/01/01/idcf-linuxmint17-android-sdk-cordova/cordova-emulate.png %}




