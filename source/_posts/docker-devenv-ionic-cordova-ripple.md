title: "Docker開発環境のIonicとRippleをnpmからインストールして使う"
date: 2015-01-05 21:26:12
tags:
 - DockerDevEnv
 - Ionic
 - Cordova
 - Ripple
 - npm
 - AndroidSDK
 - HTML5HybridMobileApps
description: IonicのDocker開発環境のエミュレータにApache Rippleをnpmからインストールして使います。以前はChromeエクステンションのRipple Emulator (Beta)として提供されていました。Ripple is Reborn!によるとPhoneGap 2.6のあたりから動作が不安定になったようです。それまではBlackberryの開発チームによりメンテナンスされていましたが、2013年の11月に新しくApache Rippleとして生まれ変わっています。
---

[IonicのDocker開発環境](/2014/12/30/docker-devenv-ionic-cordova/)のエミュレータに[Apache Ripple](http://ripple.incubator.apache.org/)をnpmからインストールして使います。以前はChromeエクステンションの[Ripple Emulator (Beta)](https://chrome.google.com/webstore/detail/ripple-emulator-beta/geelfhphabnejjhdalkjhgipohgpdnoc)として提供されていました。[Ripple is Reborn!](http://www.raymondcamden.com/2013/11/05/Ripple-is-Reborn)によるとPhoneGap 2.6のあたりから動作が不安定になったようです。それまではBlackberryの開発チームによりメンテナンスされていましたが、2013年の11月に新しくApache Rippleとして生まれ変わっています。

<!-- more -->

### Linux Mint 17.1 MATEの環境を使う

最初に[Linux Mint 17.1 MATE](https://masato.github.io/2015/01/01/idcf-linuxmint17-android-sdk-cordova/)に構築したGUIのCordova開発環境へRippleをインストールして確認します。[ripple-emulator](https://www.npmjs.com/package/ripple-emulator)をグローバルにインストールします。

``` bash
$ npm install -g cordova ionic
$ npm install -g ripple-emulator
```

Rippleのバージョンは0.9.24です。

```
$ ripple version
0.9.24
```

### Ionicのサンプル作成

テスト用にIonicのサンプルアプリを作成します。

``` bash
$ mkdir ~/ionic_apps
$ cd !$
$ ionic start ionicTestApp tabs
$ cd ionicTestApp 
$ ionic platform add android
```

### Rippleの起動

`ripple emulate`コマンドを実行するとデフォルトのブラウザが自動的に起動します。`--path`フラグはビルド前の`www`ディレクトリを指定します。

``` bash
$ cd ~/ionic_apps/ionicTestApp/
$ ripple emulate --path www
```

URLは次のようにクエリ文字列に`?enableripple=cordova-3.0.0`が付いています。
http://localhost:4400?enableripple=cordova-3.0.0

{% img center /2015/01/05/docker-devenv-ionic-cordova-ripple/ripple-mint-17.png %}

クエリ文字列を外して、http://localhost:4400をブラウザから開くと通常のionicのアプリになります。

### Docker開発環境で確認

Dockerイメージは[masato/ionic-cordova](https://registry.hub.docker.com/u/masato/ionic-cordova/)を使います。使い捨ての開発環境コンテナを起動します。

``` bash
$ docker run -it --rm --name ionic-env masato/ionic-cordova  bash
```

dockerユーザーにスイッチして、予め作成してあるIonicのサンプルアプリのディレクトリに移動してRippleを起動します。

``` bash
$ su - docker
$ cd /data/ionicTestApp
$ nvm use v0.10
Now using node v0.10.35
$ ripple emulate --path www
```

### ngrok

開発環境コンテナのIPアドレスを確認して、ngrokでトンネルします。

``` bash
$ docker inspect --format="&#123;&#123; .NetworkSettings.IPAddress }}" ionic-env
172.17.4.93
$ docker run -it --rm wizardapps/ngrok:latest ngrok 172.17.4.93:4400
```

先ほどと同様にクエリ文字列に`?enableripple=cordova-3.0.0`を指定してブラウザから開きます。
http://6e7e14f6.ngrok.com/?enableripple=cordova-3.0.0

{% img center /2015/01/05/docker-devenv-ionic-cordova-ripple/ripple-docker.png %}


### まとめ

Cordovaのエミュレーターは起動に時間がかかります。Rippleは起動も速く、GUI開発環境がなくてエミュレータで確認ができるのでDocker開発環境に便利です。
