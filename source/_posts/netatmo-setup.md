title: "Netatmo ウェザーステーションのAPIを使ってみる - Part1: Node.jsのサンプル"
date: 2015-08-29 21:28:12
categories:
 - IoT
tags:
 - Netatmo
 - IoT
 - Nodejs
 - API
 - lodash
 - myThings
---

Netatmo ウェザーステーションを使うと、屋内や屋外に設置したモジュールから簡単に環境データを計測して、スマホやデスクトップブラウザのダッシュボードから確認することができます。計測できるデータは気温、湿度、気圧、二酸化炭素濃度、騒音などです。[myThings](http://mythings.yahoo.co.jp/)にも対応しています。オフィシャルでは[API](https://dev.netatmo.com/doc/)を使うためのSDKはPHP、Objective-C、Windows8、Android用が用意されています。コミュニティーベースですが、Node.js用のライブラリもGitHubにリポジトリがあったので、こちらを使ってみようと思います。

<!-- more -->

## Netatmoのセットアップ

[室内＆屋外の気温・湿度・CO2等を計測して詳細な天気情報を出してくれる「netatmo ウェザーステーション＆雨量計」レビュー](http://gigazine.net/news/20140829-netatmo/)が写真入りで解説されているので参考にさせていただきます。また、スマホアプリのガイドも丁寧でわかりやすいので、アプリを操作していくだけでも簡単に設定していくことができます。

### サインアップ

iPhoneの場合は[Weather Station by Netatmo](https://itunes.apple.com/ja/app/netatmo-weather-station/id532538499)、Androidは[ステーション Netatmo](https://play.google.com/store/apps/details?id=com.netatmo.netatmo&hl=ja)をインストールして、Netatmoのセットアップetatmoにサインアップします。

### 屋外モジュールとステーションの設置

屋外モジュールに電池ををセットします。ステーションはMicro USBケーブルで電源プラグに接続します。ステーションの上部をタッチして側面が青色に点滅させます。

### ステーションのWi-Fi設定

スマホアプリからステーションにBluetoothでペアリングしたあと、ステーションにWi-Fiのセットアップを行います。接続に成功するとステーションをNetatmoに登録して終了します。


### デスクトップブラウザ

デスクトップブラウザから[ログイン](https://auth.netatmo.com/access/login)します。[ウェザーステーション](https://my.netatmo.com/app/station)のページを開くと以下のように表示されます。


![netatmo-station.png](/2015/08/29/netatmo-setup/netatmo-station.png)

## APIを使う準備

APIを使ったサンプルは、[Netatmo ウェザーステーションを買ってみたので Node.js でいじってみた](http://tips.hecomi.com/entry/2014/04/14/211252)を参考にさせていただきました。試してみたいNode.jsのライブラリの例なので助かりました。

### アプリの登録

[Netatmo Developers](https://dev.netatmo.com/)のページを開いてログインします。上メニューに[DOCUMENTATION](https://dev.netatmo.com/doc/)と[CREATE AN APPLICATION](https://dev.netatmo.com/dev/createapp)などが表示されています。CREATE AN APPLICATIONのページを開き「Name」と「Description」に適当な名前をつけて登録すると、Client idとClient secretが発行されます。APIではこのidとsecretを使います。

![netatmo-apikey.png](/2015/08/29/netatmo-setup/netatmo-apikey.png)


## Node.jsのサンプル

Node.jsの[netatmo](https://github.com/karbassi/netatmo)を利用してAPIを使ったサンプルを書いてみます。リポジトリは[こちら](https://github.com/masato/docker-netatmo)です。今回もベースイメージに[io.js](https://hub.docker.com/_/iojs/)の[3.2](https://github.com/nodejs/docker-iojs/blob/68a13ab0f190d079b484c67b9eb266ce999420b0/3.2/Dockerfile)を利用してDocker Composeから構成管理します。

### docker-compose.yml

docker-compose.yml.defaultファイルをdocker-compose.ymlにリネームしてから環境変数を編集します。

* CLIENT_ID: 作成したアプリのClient id
* CLIENT_SECRET: 作成したアプリのClient secret
* USERNAME: netatmoにログインするユーザー名
* PASSWORD: netatmoにログインするパスワード名
* INTERVAL: 1800000(30分)、 netatmoからデータを取得する感覚

```yml ~/node_apps/docker-netatmo/docker-compose.yml.default
netatmo:
  build: .
  volumes:
    - .:/app
    - /etc/localtime:/etc/localtime:ro
  environment:
    - CLIENT_ID=
    - CLIENT_SECRET=
    - USERNAME=
    - PASSWORD=
    - INTERVAL=1800000
  ports:
    - 3033:3000
```

### app.js

Netatomoには[GETMEASURE](https://dev.netatmo.com/doc/methods/getmeasure)というAPIもありますが、デバイス(ステーション)とモジュール(屋外モジュール)のどちらかしかデータを取得できないようです。親子関係を紐付けるidをつかって一つのJSONオブジェクトにまとめてみました。

また、モジュールのtypeフィールドが`NAModule4`の場合は追加の屋内モジュールになります。こちらはCO2を検出できるので追加します。

```js ~/node_apps/docker-netatmo/app.js
"use strict";

var netatmo = require('netatmo'),
    async = require('async'),
    _ = require('lodash'),
    moment = require('moment-timezone'),
    api = new netatmo({
        'client_id': process.env.CLIENT_ID,
        'client_secret': process.env.CLIENT_SECRET,
        'username': process.env.USERNAME,
        'password': process.env.PASSWORD,
    });

function getMyDevices(deviceIds, devices, modules, callback) {
    var myDevices = _.map(deviceIds, function(id) {
        return {
            device: _.find(devices, function(device) {
                return id == device._id;
            }),
            modules: _.filter(modules, function(module) {
                return id == module.main_device;
            })
        };
    });
    callback(null, myDevices);
}

function toJST(time_utc) {
  return moment(time_utc, 'X').tz("Asia/Tokyo").format();
}

function pluckDevice(device) {
    return {
        module_name: device.module_name,
        time_utc: toJST(device.dashboard_data.time_utc),
        noise: device.dashboard_data.Noise,
        temperature: device.dashboard_data.Temperature,
        humidity: device.dashboard_data.Humidity,
        pressure: device.dashboard_data.Pressure,
        co2: device.dashboard_data.CO2
    };
}

function pluckModule(module) {
    var retval = {
        module_name: module.module_name,
        time_utc: toJST(module.dashboard_data.time_utc),
        temperature: module.dashboard_data.Temperature,
        humidity: module.dashboard_data.Humidity
    };
    // for the additionnal indoor module
    // https://dev.netatmo.com/doc/methods/devicelist
    if (module.type == 'NAModule4'){
        retval[co2] = module.dashboard_data.CO2;
    }
    return retval;
}

function resultsOut(results) {
    _.forEach(results, function(result) {
        console.log(pluckDevice(result.device));
        _.forEach(result.modules, function(module) {
            console.log(pluckModule(module));
        });
    });
}

function measure(callback) {
    async.waterfall([
        function(callback) {
            api.getUser(callback);
        },
        function(user, callback) {
            api.getDevicelist(function(err, devices, modules) {
                if (err) return callback(err);
                getMyDevices(user.devices,
                             devices, modules,
                             callback);
            });
        }], function(err, results) {
            if (err) return console.log(err);
            resultsOut(results);
            console.log('----');
            setTimeout(callback, process.env.INTERVAL);
        });
}

async.forever( measure, function(err) {
    if(err) console.log(err);
});
```

### 実行

プログラムを実行すると環境変数INTERVALに定義した間隔(1800000ミリ秒=30分)でNetatomoからデータを取得して標準出力します。本体とモジュールのセンシング時間は完全に一致はしていないようです。

```bash
$ docker-compose up netatmo
...
netatmo_1 | npm info it worked if it ends with ok
netatmo_1 | npm info using npm@2.13.3
netatmo_1 | npm info using node@v3.2.0
netatmo_1 | npm info prestart netatmo-docker@0.0.1
netatmo_1 | npm info start netatmo-docker@0.0.1
netatmo_1 |
netatmo_1 | > netatmo-docker@0.0.1 start /app
netatmo_1 | > node app.js
netatmo_1 |
netatmo_1 | { module_name: 'リビング',
netatmo_1 |   time_jst: '2015-08-31T09:34:30+09:00',
netatmo_1 |   noise: 38,
netatmo_1 |   temperature: 24.5,
netatmo_1 |   humidity: 84,
netatmo_1 |   pressure: 1081.7,
netatmo_1 |   co2: 395 }
netatmo_1 | { module_name: 'ベランダ',
netatmo_1 |   time_jst: '2015-08-31T09:34:28+09:00',
netatmo_1 |   temperature: 22.9,
netatmo_1 |   humidity: 86 }
netatmo_1 | ----
```