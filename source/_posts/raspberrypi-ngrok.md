title: "Raspberry Piをngrokで公開する - Part1: Expressをpm2から起動する"
date: 2015-08-04 13:52:42
categories:
 - IoT
tags:
 - RaspberryPi
 - ngrok
 - pm2
 - Nodejs
description: Raspberry Piで起動しているサービスにインターネットから直接アクセスしたい場合があります。開発中にプライベートネットワークのサーバーをトンネルしてリモートからテストするとき、ngrokを便利に使っています。同じやり方でRaspberry PiにNode.jsラッパーのnorokをインストールしてExpressのHello Worldをリモートから表示してみます。
---

Raspberry Piで起動しているサービスにインターネットから直接アクセスしたい場合があります。開発中にプライベートネットワークのサーバーをトンネルしてリモートからテストするとき、[ngrok](https://ngrok.com/)を便利に使っています。同じやり方でRaspberry PiにNode.jsラッパーの[norok](https://github.com/bubenshchykov/ngrok)をインストールしてExpressのHello Worldをリモートから表示してみます。

<!-- more -->


## ngrok 0.1.99

今回はnpmからインストールします。[ngrok](https://github.com/bubenshchykov/ngrok)のバージョンは`0.1.99`です。norokはすでに2.x系がリリースされています。GitHubからcloneしてインストールすると2.xに対応したバージョンを使えます。

## Node.js

[node-arm](http://node-arm.herokuapp.com/)からdebパッケージをダウンロードしてNode.jsをインストールします。

```bash
$ wget http://node-arm.herokuapp.com/node_latest_armhf.deb
$ sudo dpkg -i node_latest_armhf.deb
```

## プロジェクト

プロジェクトを作成します。作業したリポジトリは[こちら](https://github.com/masato/node-hello-ngrok-pm2)です。

```bash
$ mkdir -p ~/node_apps/node-hello-ngrok-pm2
$ cd !$
```

### package.json

package.jsonのひな形を作成しておきます。

```json ~/node_apps/node-hello-ngrok-pm2/package.json
{
  "name": "node-hello-ngrok-pm2",
  "description": "node-hello-ngrok-pm2",
  "version": "0.0.1",
  "private": true,
  "scripts": {
    "start": "node app.js"
  }
}
```

[ngrok](https://www.npmjs.com/package/ngrok)、[Express](https://www.npmjs.com/package/express)、[dotenv](https://www.npmjs.com/package/dotenv)をインストールします。

```bash
$ npm install ngrok express dotenv --save
```

package.jsonのdependenciesにインストールしたパッケージが追記されました。

```json ~/node_apps/node-hello-ngrok-pm2/package.json
{
  "name": "node-hello-ngrok-pm2",
  "description": "node-hello-ngrok-pm2",
  "version": "0.0.1",
  "private": true,
  "scripts": {
    "start": "node app.js"
  },
  "dependencies": {
    "dotenv": "^1.2.0",
    "express": "^4.13.3",
    "ngrok": "^0.1.99"
  }
}
```

[ngrok](https://ngrok.com/)のバイナリもnpmからインストールされます。バージョンを確認します。

```bash
$ ~/node_apps/node-hello-ngrok-pm2/node_modules/ngrok/bin/ngrok version
1.7
```

### app.js

app.jsを作成します。Expressを起動してngrokで公開する単純なプログラムです。

```js ~/node_apps/node-hello-ngrok-pm2/app.js
var ngrok = require('ngrok'),
    express = require('express');

require('dotenv').load();
var app = express();

app.get('/', function(req, res) {
    res.send('Hello World!');
});

var server = app.listen(process.env.EXPRESS_PORT, function() {
    var host = server.address().address,
        port = server.address().port;

    console.log('listening at http://%s:%s', host, port);
    ngrok.connect({
        port: port
    },
    function (err, url) {
        console.log(url);
    });
});
```

Expressのポート番号は環境変数として.envに記述しておきます。

```bash ~/node_apps/node-node-hello-ngrok-pm2-pm2/.env
EXPRESS_PORT=3000
```

テストのため直接`npm start`を実行して起動します。

```bash
$ npm start

> node-hello-ngrok-pm2@0.0.1 start /home/pi/node_apps/node-hello-ngrok-pm2
> node app.js

listening at http://0.0.0.0:3000
https://26802xxx.ngrok.com
```

標準出力されたngrokのURLを使い、リモートからRaspberry PiのExpressにアクセスできるようになりました。

```bash
$ curl https://26802xxx.ngrok.com
Hello World!
```

## pm2

[pm2](https://github.com/Unitech/pm2)を使ってデモナイズします。npmパッケージはグローバルにインストールします。

### インストール

```bash
$ sudo npm install pm2 -g
```

Raspberry Piの再起動後も、pm2に登録したプロセスが起動するようにinitスクリプトを作成します。

```bash
$ sudo env PATH=$PATH:/usr/local/bin pm2 startup -u pi
[PM2] Spawning PM2 daemon
[PM2] PM2 Successfully daemonized
[PM2] Generating system init script in /etc/init.d/pm2-init.sh
[PM2] Making script booting at startup...
[PM2] -linux- Using the command:
      su -c "chmod +x /etc/init.d/pm2-init.sh && update-rc.d pm2-init.sh defaults"
update-rc.d: using dependency based boot sequencing
[PM2] Done.
$ pm2 save
[PM2] Dumping processes
```

`/etc/init.d/pm2-init.sh`が作成されました。エディタで開き`PM2_HOME`を編集します。

```bash /etc/init.d/pm2-init.sh
#export PM2_HOME="/root/.pm2"
export PM2_HOME="/home/pi/.pm2"
```

`PM2_HOME`を`/home/pi/.pm2`に変更しないとinitスクリプトの起動時にエラーがでます。

```bash
$ sudo /etc/init.d/pm2-init.sh restart
Restarting pm2
fs.js:747
  return binding.mkdir(pathModule._makeLong(path),
                 ^
Error: EACCES, permission denied '/root/.pm2'
    at Error (native)
    at Object.fs.mkdirSync (fs.js:747:18)
```

### プロセスの起動

pm2からサービスを起動します。

```bash
$ cd ~/node_apps/node-hello-ngrok-pm2
$ pm2 start app.js
[PM2] Starting app.js in fork_mode (1 instance)
[PM2] Done.
┌──────────┬────┬──────┬──────┬────────┬─────────┬────────┬────────────┬──────────┐
│ App name │ id │ mode │ pid  │ status │ restart │ uptime │ memory     │ watching │
├──────────┼────┼──────┼──────┼────────┼─────────┼────────┼────────────┼──────────┤
│ app      │ 0  │ fork │ 2593 │ online │ 0       │ 0s     │ 8.344 MB   │ disabled │
└──────────┴────┴──────┴──────┴────────┴─────────┴────────┴────────────┴──────────┘
 Use `pm2 show <id|name>` to get more details about an app
```

標準出力されているngrokのURLを使って、curlからアクセスします。

```bash
$ pm2 logs --lines 2 app
[PM2] Tailing last 2 lines for [app] process

app-0 (out): listening at http://0.0.0.0:3000
app-0 (out): https://2724bxxx.ngrok.com
$ curl https://2724bxxx.ngrok.com
Hello World!
```

### restart

initスクリプトからpm2を再起動したあとのアプリの起動をテストします。

```bash
$ sudo /etc/init.d/pm2-init.sh restart
Restarting pm2
[PM2] Dumping processes
[PM2] Deleting all process
[PM2] deleteProcessId process id 0
┌──────────┬────┬──────┬─────┬────────┬─────────┬────────┬────────┬──────────┐
│ App name │ id │ mode │ pid │ status │ restart │ uptime │ memory │ watching │
└──────────┴────┴──────┴─────┴────────┴─────────┴────────┴────────┴──────────┘
 Use `pm2 show <id|name>` to get more details about an app
[PM2] Stopping PM2...
[PM2][WARN] No process found
[PM2] All processes have been stopped and deleted
[PM2] PM2 stopped
Starting pm2
[PM2] Spawning PM2 daemon
[PM2] PM2 Successfully daemonized
[PM2] Resurrecting
Process /home/pi/node_apps/node-hello-ngrok-pm2/app.js launched
┌──────────┬────┬──────┬──────┬────────┬─────────┬────────┬─────────────┬──────────┐
│ App name │ id │ mode │ pid  │ status │ restart │ uptime │ memory      │ watching │
├──────────┼────┼──────┼──────┼────────┼─────────┼────────┼─────────────┼──────────┤
│ app      │ 0  │ fork │ 2779 │ online │ 0       │ 0s     │ 13.176 MB   │ disabled │
└──────────┴────┴──────┴──────┴────────┴─────────┴────────┴─────────────┴──────────┘
 Use `pm2 show <id|name>` to get more details about an app
```

logからngrokのURLを出力します。

```bash
$ pm2 logs --lines 2 app
[PM2] Tailing last 2 lines for [app] process

app-0 (out): listening at http://0.0.0.0:3000
app-0 (out): https://2724bxxx.ngrok.com
```

リモートからRaspberry PiのExpressにアクセスできました。

```bash
$ curl https://2724bxxx.ngrok.com
Hello World!
```

最後にOSをrebootした後もサービスが起動していることを確認しておきます。

```bash
$ sudo reboot
```

