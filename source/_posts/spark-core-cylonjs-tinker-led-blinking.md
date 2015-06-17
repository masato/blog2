title: "Spark CoreのTinkerファームウェアでREST API/spark-cli/Cylon.jsでLチカする"
date: 2015-03-02 14:05:52
tags:
 - SparkCore
 - SparkCloud
 - Cylonjs
 - Lチカ
description: Spark CoreにはデフォルトでTinkerファームウェアがインストールされています。前回はAndroidなどのTinkerアプリなどから操作するために使いました。TinkerファームウェアはArduinoのFirmataファームウェアと異なり、インターネット上のSpark Cloudを経由して通信します。そのため直接ホストマシンと接続する必要がありません。リモートにあるSpark CoreをREST APIなどから操作することができます。自分のクラウド上の仮想マシンからNode.jsのプログラムを実行することも可能です。
---

Spark Coreにはデフォルトで[Tinkerファームウェア](http://docs.spark.io/tinker/#tinkering-with-tinker-the-tinker-firmware)がインストールされています。[前回](/2015/03/01/spark-core-led-blinking/)はAndroidなどの[Tinkerアプリ](https://play.google.com/store/apps/details?id=io.spark.core.android&hl=ja)などから操作するために使いました。TinkerファームウェアはArduinoの[Firmata](http://arduino.cc/en/reference/firmata)ファームウェアと異なり、インターネット上のSpark Cloudを経由して通信します。そのため直接ホストマシンと接続する必要がありません。リモートにあるSpark CoreをREST APIなどから操作することができます。自分のクラウド上の仮想マシンからNode.jsのプログラムを実行することも可能です。

<!-- more -->

## ブレッドボードとLEDの準備

[Spark-Core-LED](http://mchobby.be/wiki/index.php?title=Spark-Core-LED)を参考にしてブレッドボードにLEDと抵抗を配線します。

![Spark-Core-Led-01.jpg](/2015/03/02/spark-core-cylonjs-tinker-led-blinking/Spark-Core-Led-01.jpg)

D0のGPIOを使ってLチカします。

## spark-cli

[spark-cli](https://github.com/spark/spark-cli)を自分のクラウド上にデプロイしてSpark CoreをLチカしてみます。

### Dockerコンテナの用意

適当なクラウドに仮想マシンとDockerをインストールしてプロジェクトを作成します。今回はIDCFクラウドの仮想マシンを使いました。最初にプロジェクトを作成します。

``` bash
$ mkdir ~/docker_apps/spark_apps/my-spark-cli
$ cd !$
```

spark-cliはグローバルインストールするので、package.jsonには依存パッケージとして定義しません。

```json package.json
{
  "name": "my-spark-cli",
  "version": "0.0.1",
  "private": true
}
```

Dockerfileを作成してイメージをビルドします。ベースイメージには[google/nodejs-runtime](https://registry.hub.docker.com/u/google/nodejs-runtime/)を指定します。

``` bash
$ cat <<EOF > Dockerfile
FROM google/nodejs-runtime
RUN npm install -g spark-cli
ENTRYPOINT ["/bin/bash"]
EOF
$ docker pull google/nodejs-runtime
$ docker build -t my-spark-cli .
$ docker run --rm --name spark-cli -it my-spark-cli
```

### ログイン

CLIを使ってSpark Cloudにログインします。テンポラリのaccess_tokenを取得できるので以下の操作で使います。

``` bash
$ spark cloud login
Could I please have an email address?  ma6ato@gmail.com
and a password?  *********
Got an access token! xxx
logged in!  { '0': 'xxx' }
Using the setting "access_token" instead
```

### CLIからLチカ

`spark cloud list`を実行すると接続しているSpark Coreの一覧を取得できます。`xxx`のところに実際にはdevice_idが入っています。

``` bash
$ spark cloud list
Checking with the cloud...
Retrieving cores... (this might take a few seconds)
ninja_mighty (xxx) is online
  Functions:
    int digitalread(String args)
    int digitalwrite(String args)
    int analogread(String args)
    int analogwrite(String args)
```

このデバイスで使える関数の一覧が表示されます。digitalwriteを使いLチカしてみます。

D0のLEDを点灯(HIGH)します。成功すると1が返ります。

``` bash
$ spark call ninja_mighty digitalwrite D0,HIGH
1
```

D0のLEDを消灯(LOW)します。

``` bash
$ spark call ninja_mighty digitalwrite D0,LOW
１
```

## REST APIからLチカ

spark-cliで確認した{access_token}と{device_id}を使って[REST API](http://docs.spark.io/tinker/#tinkering-with-tinker-the-tinker-api)を試してみます。

D0のLEDを点灯(HIGH)します。

``` bash
$ curl https://api.spark.io/v1/devices/{device_id}/digitalwrite \
  -d access_token={access_token} \
  -d params=D0,HIGH
{
  "id": "xxx",
  "name": "ninja_mighty",
  "last_app": null,
  "connected": true,
  "return_value": 1
}
```

D0のLEDを消灯(LOW)します。

``` bash
$ curl https://api.spark.io/v1/devices/{device_id}/digitalwrite \
  -d access_token={access_token} \
  -d params=D0,LOW
{
  "id": "53ff6d066667574821362467",
  "name": "ninja_mighty",
  "last_app": null,
  "connected": true,
  "return_value": 1
}
```

## Cylon.jsでLチカ

[Cylon.js](http://cylonjs.com/documentation/platforms/spark/)の[cylon-spark](https://github.com/hybridgroup/cylon-spark)アダプタを使い、Node.jsのプログラムからLチカします。

### Dockerコンテナの用意

Dockerをインストールした仮想マシンにプロジェクトを作成します。

``` bash
$ mkdir ~/docker_apps/spark_apps/led-blinking
$ cd !$
```

package.jsonに必要なパッケージを定義します。

```json ~/docker_apps/spark_apps/led-blinking/package.json
{
  "name": "spark-led-blinking",
  "version": "0.0.1",
  "private": true,
  "dependencies": {
    "cylon-spark": "0.18.0"
  },
  "scripts": {"start": "node app.js"}
}
```

app.jsのメインプログラムを書きます。access_tokenとdevice_idはDokerコンテナの起動時に環境変数として渡します。

```js ~/docker_apps/spark_apps/led-blinking/app.js
var Cylon = require('cylon');

Cylon.robot({
  connections: {
    spark: { adaptor: 'spark',
             accessToken: process.env.ACCESS_TOKEN,
             deviceId: process.env.DEVICE_ID
           }
  },
  devices: {
    led: { driver: 'led', pin: 'D0'}
  },

  work: function(my) {
    every((1).second(), function() {my.led.toggle()});
  }
}).start();
```

Dockerfileを作成してイメージをビルドします。

``` bash
$ echo FROM google/nodejs-runtime > Dockerfile
$ docker pull google/nodejs-runtime
$ docker build -t spark-led-blinking .
```

### Lチカの実行

Dockerコンテナを起動します。{device_id}はspike-cliで取得した値と同じですが、{access_token}の値はテンポラリのようです。[Spark Build](https://www.spark.io/build)にログインして、Settingsメニューからアクセストークンを確認します。

![spark-ide-access-token.png](/2015/03/02/spark-core-cylonjs-tinker-led-blinking/spark-ide-access-token.png)

{device_id}もSpark BuildのCoresメニューから確認できます。

![spark-ide-device-id.png](/2015/03/02/spark-core-cylonjs-tinker-led-blinking/spark-ide-device-id.png)

`docker run`の`-e`フラグに{access_token}と{device_id}を指定してコンテナを起動します。package.jsonの`scripts`ディレクティブに指定したapp.jsを実行してLチカが始まります。

``` bash
$ docker run --rm  --name led-blinking \
  -e ACCESS_TOKEN={access_token} \
  -e DEVICE_ID={device_id} \
  -it spark-led-blinking 

> spark-led-blinking@0.0.1 start /app
> node app.js

I, [2015-03-02T05:03:37.022Z]  INFO -- : [Robot 57708] - Initializing connections.
I, [2015-03-02T05:03:37.229Z]  INFO -- : [Robot 57708] - Initializing devices.
I, [2015-03-02T05:03:37.232Z]  INFO -- : [Robot 57708] - Starting connections.
I, [2015-03-02T05:03:38.186Z]  INFO -- : [Robot 57708] - Starting devices.
I, [2015-03-02T05:03:38.186Z]  INFO -- : [Robot 57708] - Working.
```




