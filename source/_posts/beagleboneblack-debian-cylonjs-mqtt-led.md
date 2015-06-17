title: "BeagleBone BlackのDebianでCylon.jsを使いMQTTを受信してLチカする"
date: 2015-02-22 10:37:46
tags:
 - BeagleBoneBlack
 - Cylonjs
 - MQTT
 - Lチカ
description: BeagleBone BlackでMQTTのメッセージを受信して標準出力するサンプルを作成しました。次はもう少し複雑な処理を実装してみます。Cylon.jsのBeagleBone Blackアダプタを使いメッセージを受信したらLチカするだけなのですが、kernelバージョンの問題やCylon.jsのDSLの変更など、嵌まりどころが結構ありました。
---

BeagleBone BlackでMQTTのメッセージを受信して標準出力する[サンプル](/2015/02/21/ifttt-do-button-wordpress-recipe/)を作成しました。次はもう少し複雑な処理を実装してみます。Cylon.jsのBeagleBone Blackアダプタを使いメッセージを受信したらLチカするだけなのですが、kernelバージョンの問題やCylon.jsのDSLの変更など、嵌まりどころが結構ありました。

<!-- more -->

## ファームウェアのDebian 7.5

BeagleBone BlackのファームウェアはDebian 7.5をeMMCに焼いてあります。

``` bash
$ cat /etc/debian_version 
7.5
```

3.8.13 kernelを使っているので、BoneScriptがサポートしているバージョンです。Cylon.jsの[cylon-beaglebone](https://github.com/hybridgroup/cylon-beaglebone)アダプターを使ったLチカにも対応していました。

``` bash
$ uname -a
Linux beaglebone 3.8.13-bone50 #1 SMP Tue May 13 13:24:52 UTC 2014 armv7l GNU/Linux
```

Node.jsの環境も確認しておきます。

``` bash
$ which node
/usr/bin/node
$ node -v
v0.10.25
$ which npm
/usr/bin/npm
$ npm -v
1.3.10
```

## インストール

package.jsonを作成し必要なパッケージを定義します。

```json package.json
{
  "name": "mqtt-led",
  "version": "0.0.1",
  "private": true,
  "dependencies": {
    "cylon-beaglebone": "0.14.0",
    "cylon-mqtt": "0.4.0"
  },
  "scripts": {"start": "node app.js"}
}
```

システムクロックの時間がずれていると`npm install`で`CERT_NOT_YET_VALID`が発生してエラーになります。BeagleBone Blackの電源を入れたあとはntpdateで時間合わせをします。

``` bash
$ sudo ntpdate -b -s -u pool.ntp.org
$ npm install
```

## MQTTのsubscribeでLチカする

### Cylon.jsのMQTTドライバーの注意

[cylon-mqtt](https://github.com/hybridgroup/cylon-mqtt/)の`Arduino Blink`を参考にしてコードを書いていたのですが、このサンプルが間違っていて実行するとエラーになります。

``` bash
Toggling LED.
Message on 'toggle': toggle

/home/debian/node_apps/node_modules/cylon-gpio/lib/led.js:82
  this.connection.digitalWrite(this.pin, 1);
                  ^
TypeError: Object #<Adaptor> has no method 'digitalWrite'
    at Led.turnOn (/home/debian/node_apps/node_modules/cylon-gpio/lib/led.js:82:19)
    at Led.toggle (/home/debian/node_apps/node_modules/cylon-gpio/lib/led.js:108:10)
    at Driver.<anonymous> (/home/debian/node_apps/app.js:20:14)
```

devicesに`adaptor: 'firmata'`とありますがプロパティが間違っています。

``` js blink.js
  devices: {
    toggle: { driver: 'mqtt', topic: 'toggle', adaptor: 'mqtt' },
    led: { driver: 'led', pin: '13', adaptor: 'firmata' },
  },
```

examplesディレクトリにある[blink.js](https://github.com/hybridgroup/cylon-mqtt/blob/master/examples/blink/blink.js)の方はconnectionプロパティに正しく修正されています。

``` js blink.js
  devices: {
    toggle: { driver: "mqtt", topic: "toggle", connection: "mqtt" },
    led: { driver: "led", pin: "13", connection: "firmata" },
  },
```

Cylon.jsは宣言的に書けるのが良いところです。だたDSLが強くなると仕様変更やプロパティの意味がわかりづらくなり、デバッグに時間がかかるのが気になります。

### 自分でPub/Subして1秒間隔でLチカする

1秒間隔でMQTTにpublishします。work内でsubscribeもしているので自分でメッセージを受信してLチカします。

``` js app.js
/*eslint-env node */

var Cylon = require('cylon');

// Initialize the robot
Cylon.robot({
  connections: {
    mqtt: { adaptor: 'mqtt', host: 'mqtt://xxx.xxx.xxx.xxx:1883' },
    beaglebone: { adaptor: 'beaglebone' }
  },

  devices: {
    toggle: { driver: 'mqtt', topic: 'toggle', connection: 'mqtt' },
    led: { driver: 'led', pin: 'P8_10', connection: 'beaglebone' }
  },

  work: function(my) {
    my.toggle.on('message',function(data) {
      console.log("Message on 'toggle': " + data);
      my.led.toggle();
    });

    every((1).second(), function() {
      console.log("Toggling LED.");
      my.toggle.publish('toggle');
    });
  }
}).start();
```

BeagleBone BlackのLチカするにはroot権限が必要です。

``` bash
$ sudo npm start   

> mqtt-led@0.0.1 start /home/debian/node_apps
> node app.js

I, [2015-02-22T02:18:03.310Z]  INFO -- : Initializing connections.
I, [2015-02-22T02:18:06.156Z]  INFO -- : Initializing devices.
I, [2015-02-22T02:18:06.195Z]  INFO -- : Starting connections.
I, [2015-02-22T02:18:06.261Z]  INFO -- : Starting devices.
I, [2015-02-22T02:18:06.271Z]  INFO -- : Working.
Toggling LED.
Message on 'toggle': toggle
Toggling LED.
Message on 'toggle': toggle
Toggling LED.
Message on 'toggle': toggle
Toggling LED.
Message on 'toggle': toggle
```


### IFTTTのDo ButtonからLチカする

[前回](/2015/02/21/ifttt-do-button-wordpress-recipe/)作成したIFTTTのDo Buttonをトリガーにしたサンプルを使います。メッセージを受信すると3秒間だけLチカします。

``` js app.js
/*eslint-env node */

var Cylon = require('cylon');

// Initialize the robot
Cylon.robot({
  connections: {
    mqtt: { adaptor: 'mqtt', host: 'mqtt://xxx.xxx.xxx.xxx:1883' },
    beaglebone: { adaptor: 'beaglebone' }
  },

  devices: {
    ifttt: { driver: 'mqtt', topic: 'ifttt/bbb', connection: 'mqtt' },
    led: { driver: 'led', pin: 'P8_10', connection: 'beaglebone' }
  },

  work: function(my) {
    my.ifttt.on("message",function(data) {
      console.log("Message on 'ifttt': " + data);
      var timer = setInterval(my.led.toggle, 200);
      setTimeout(function(){ clearInterval(timer)},3000);
    });
  }
}).start();
```
 
AndroidにインストールしたDo Buttonアプリのボタンをタップしてトリガーを起動します。

``` bash
$ sudo npm start

> mqtt-led@0.0.1 start /home/debian/node_apps
> node app.js

I, [2015-02-22T02:06:06.841Z]  INFO -- : Initializing connections.
I, [2015-02-22T02:06:09.746Z]  INFO -- : Initializing devices.
I, [2015-02-22T02:06:09.784Z]  INFO -- : Starting connections.
I, [2015-02-22T02:06:09.849Z]  INFO -- : Starting devices.
I, [2015-02-22T02:06:09.860Z]  INFO -- : Working.
Message on 'ifttt': {"username":"username","password":"password","title":"","description":"<div><img src='http://maps.google.com/maps/api/staticmap?center=35.xxxxxx,139.xxxxxx&zoom=19&size=640x440&scale=1&maptype=roadmap&sensor=false&markers=color:red%7C35.xxxxxx,139.xxxxxx' style='max-width:600px;' /><br/><div>Do Button pressed on February 22, 2015 at 11:06AM http://ift.tt/1LzZoYL</div></div>","tags":{"string":"Do Button"},"post_status":"publish"}
```

