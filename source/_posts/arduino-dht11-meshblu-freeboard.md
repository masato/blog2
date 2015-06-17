title: "ArduinoからDHT11の温度と湿度データをMQTTで送信してfreeboardに表示する"
date: 2015-04-27 16:02:52
tags:
 - freeboard
 - Arduino
 - DHT11
 - センサー
 - Nodejs
 - Docker
 - DockerCompose
description: ArduinoとDHT11デジタル温度センサーからMeshbluのMQTTブローカーにメッセージをpublishするサンプルを作成しました。またWebブラウザから直接メッセージを取得することもできました。これらを組み合わせてリアルタイムにセンシングデータをモニタリングするダッシュボードをfreeboardで作ってみます。
---

[ArduinoとDHT11デジタル温度センサー](/2015/04/20/arduino-dht11-mqtt-wr702n-meshblu/)からMeshbluのMQTTブローカーにメッセージをpublishするサンプルを作成しました。また[Webブラウザ](/2015/04/26/meshblu-mqtt-websocket-javascript-browser/)から直接メッセージを取得することもできました。これらを組み合わせてリアルタイムにセンシングデータをモニタリングするダッシュボードを[freeboard](https://github.com/Freeboard/freeboard)で作ってみます。

<!-- more -->


## データソース

### freeboard-mqtt

最初は[freeboard-mqtt](https://github.com/alsm/freeboard-mqtt)を参考にしてPahoの[JavaScript Client](http://eclipse.org/paho/clients/js/)をfreeboardの`external_scripts`に指定する方法を試しました。MeshbluのMQTTブローカーだとうまく動作しません。

### freeboardのMeshbluデータソース

freeboardのソースコードを読んでいると[freeboard.datasources.js](https://github.com/Freeboard/freeboard/blob/master/plugins/freeboard/freeboard.datasources.js)にMeshbluのデータソースが追加されていました。オフィシャルでサポートされているのでうまく行きそうです。


## Arduino Unoの準備

スケッチは[前回作成](/2015/04/20/arduino-dht11-mqtt-wr702n-meshblu/)作成したコードを使います。DHT11のセンサーはArduinoのPD2に配線します。 WR702NモバイルルーターとEthernetでつなぎインターネットに接続します。


## freeboardのインストール

### プロジェクト

パブリックの[freeboard](http://freeboard.github.io/freeboard/)にはまだMeshbluのデータソースが追加されていません。ローカルのDockerでfeeboardを起動します。ApachやNginxでも構いませんが[テスト用](/2015/04/21/docker-node-static-web-server/)に使っている[node-static](https://github.com/cloudhead/node-static)のコンテナとして起動します。

まず適当なディレクトリにプロジェクトを作成します。

``` bash
$ mkdir -p ~/node_apps/freeboard-static
$ cd !$
```

publicディレクトリにfreeboardのソースをgit cloneします。

``` bash
$ git clone https://github.com/Freeboard/freeboard.git public
```

以下のようなプロジェクト構成になります。publicディレクトリにfreeboadがデプロイされています。

``` bash
$ tree -L 1 .
.
├── Dockerfile
├── app.js
├── docker-compose.yml
├── package.json
└── public
```

### ソースコード

今回は[google/nodejs-runtime](https://registry.hub.docker.com/u/google/nodejs-runtime/)をベースイメージに使います。

``` bash  ~/node_apps/freeboard-static/Dockerfile
FROM google/nodejs-runtime
VOLUME /app/public
```

コンテナは1つなのでDocker Composeは不要ですが、あとで他のコンテナとオーケストレーションするために最初からDocker Composeにしておきます。

``` yaml ~/node_apps/freeboard-static/docker-compose.yml
freeboard:
  build: .
  ports:
    - 80:8080
  volumes:
    - $PWD/public:/app/public
```

package.jsonからnode-staticのインストールをします。

``` json ~/node_apps/freeboard-static/package.json
{
  "name": "node-static-app",
  "description": "node-static app",
  "version": "0.0.1",
  "private": true,
  "dependencies": {
    "node-static": "0.7.6"
  },
  "scripts": {"start": "node app.js"}
}
```

メインプログラムです。単純に`./public`ディレクトリをサーブします。

``` js ~/node_apps/freeboard-static/app.js
var static = require('node-static');
var file = new static.Server('./public');

require('http').createServer(function (request, response) {
    request.addListener('end', function () {
        file.serve(request, response);
    }).resume();
}).listen(8080);

console.log("Server running at http://localhost:8080");
```

Docker Composeからコンテナを起動します。

``` bash
$ docker-compose up
```
 

## feeboardの設定

ブラウザからfreeboardを起動しているDockerコンテナに接続します。

### Datasourceの追加

DATASOURCES > ADD > TYPE > Octobluを選択します。

UUIDとTOKENはArduinoのスケッチでMQTT publishするときに使った組み合わせを指定します。

![datasource-entry.png](/2015/04/27/arduino-dht11-meshblu-freeboard/datasource-entry.png)

### Paneの追加

ADD PANE をクリックしてあたらしくPaneを作成します。

![my-room-pane.png](/2015/04/27/arduino-dht11-meshblu-freeboard/my-room-pane.png)

### Widgetの追加

作成したPaneの+ボタンをクリックします。DHT11センサーからは湿度と温度を計測しているので、HumidityとTemperatureの2つのWidgetを作成します。


湿度のパーセントはゲージの半円グラフで表示します。VALUEの+DATASOURCEボタンをクリックしてJSONデータから取得するデータを指定します。

![humidity-gauge.png](/2015/04/27/arduino-dht11-meshblu-freeboard/humidity-gauge.png)

温度のCはテキストと折れ線グラフで表示します。

![temerature-spark.png](/2015/04/27/arduino-dht11-meshblu-freeboard/temerature-spark.png)

### 設定をJSON形式で保存

最後にこれまでの設定をJSON形式で保存してローカルにダウンロードします。ブラウザをリロードしても設定は消えてしまいます。次回は`LOAD FREEBOARD`をクリックしてJSONをアップロードします。

![save-freeboard.png](/2015/04/27/arduino-dht11-meshblu-freeboard/save-freeboard.png)

### 完成

Arduinoからは5秒間隔でDHT11センサーから計測した値をMQTT publishしています。上の工具アイコンをクリックすると編集画面になります。通常はこのダッシュボードの画面を表示します。


![freeboard-edit.png](/2015/04/27/arduino-dht11-meshblu-freeboard/freeboard-edit.png)
