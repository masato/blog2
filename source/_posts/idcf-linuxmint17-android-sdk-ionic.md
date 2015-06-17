title: "Linux Mint 17.1 MATEにIonicをインストールする"
date: 2015-01-02 11:28:49
tags:
 - IDCFクラウド
 - LinuxMint17
 - MATE
 - DockerDevEnv
 - Ionic
 - Cordova
 - AndroidSDK
description: Apache CordovaをインストールしたLinux Mint 17.1の仮想マシンにIonicもインストールしてみます。Cordovaと同じようにエミュレーターを起動します。

---

[Apache Cordovaをインストールした](/2015/01/01/idcf-linuxmint17-android-sdk-cordova/)Linux Mint 17.1の仮想マシンにIonicもインストールしてみます。Cordovaと同じようにエミュレーターを起動します。

<!-- more -->


### Ionicのインストール

npmを使いIonicをインストールします。

``` bash
$ npm install -g cordova ionic
```

### Ionicサンプルアプリの作成

作業ディレクトリを作成して、tabsテンプレートを使ったIonicのサンプルアプリを作成します。

``` bash
$ mkdir ~/ionic_apps
$ cd !$
$ ionic start ionicTestApp tabs
$ cd ionicTestApp 
$ ionic platform add android
Creating android project...
Creating Cordova project for the Android platform:
        Path: platforms/android
        Package: com.ionicframework.ionictestapp155987
        Name: ionicTestApp
        Android target: android-19
Copying template files...
Project successfully created.
Running command: /home/mshimizu/ionoc_apps/ionicTestApp/hooks/after_prepare/010_add_platform_class.js /home/mshimizu/ionoc_apps/ionicTestApp
add to body class: platform-android
Installing "com.ionic.keyboard" for android
Installing "org.apache.cordova.console" for android
Installing "org.apache.cordova.device" for android
```

ビルドが成功したらエミュレーターを起動します。

``` bash
$ ionic build
$ ionic emulate
...
BUILD SUCCESSFUL
Total time: 4 seconds
Built the following apk(s):
    /home/mshimizu/ionoc_apps/ionicTestApp/platforms/android/ant-build/CordovaApp-debug.apk
WARNING : no emulator specified, defaulting to test
Waiting for emulator...
libGL error: failed to load driver: swrast
Booting up emulator (this may take a while)........................BOOT COMPLETE
Installing app on emulator...
Using apk: /home/mshimizu/ionoc_apps/ionicTestApp/platforms/android/ant-build/CordovaApp-debug.apk
Launching application...
LAUNCH SUCCESS
```

Ioniceのエミュレーターが起動しました。

{% img center /2015/01/02/idcf-linuxmint17-android-sdk-ionic/ionic-emulate.png %}

