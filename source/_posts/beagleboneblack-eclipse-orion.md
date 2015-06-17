title: 'BeagleBone BlackのUbuntu14.04.1にEclipse Orionをインストールする'
date: 2015-02-10 00:23:31
tags:
 - Ubuntu
 - BeagleBoneBlack
 - Nodejs
 - npm
 - EclipseOrion
description: BeagleBone BlackのファームウェアにAngstromを使う場合はデフォルトでWebブラウザ上で動作するIDEのCloud9がインストールされています。前回SDカードのUbuntu14.04.1にNode.jsとnpmをインストールしました。Cloud9の代わりにEclise Orionを使ってみます。OrionはCloud Foundryと連携が強化されIBM DevOps Servicesでも採用されています。
---

BeagleBone Blackのファームウェアに[Angstrom](http://beagleboard.org/latest-images)を使う場合はデフォルトでWebブラウザ上で動作するIDEの[Cloud9](https://github.com/ajaxorg/cloud9)がインストールされています。[前回](/2015/02/09/beagleboneblack-nodejs-npm/)SDカードのUbuntu14.04.1にNode.jsとnpmをインストールしました。Cloud9の代わりに[Eclise Orion](http://eclipse.org/orion/)を使ってみます。OrionはCloud Foundryと連携が強化されIBM DevOps Servicesでも採用されています。

<!-- more -->

## Eclipse Orionのインストール

Node.jsとnpmのバージョンを確認します。

``` bash
$ node -v
v0.10.25
$ npm -v
1.3.10
```

### CERT_NOT_YET_VALIDのエラーが発生する

orionをnpmでインストールしようとすると`CERT_NOT_YET_VALID`エラーが発生しました。

``` bash
$ npm install orion --production
npm http GET https://registry.npmjs.org/orion
npm http GET https://registry.npmjs.org/orion
npm http GET https://registry.npmjs.org/orion
npm ERR! Error: CERT_NOT_YET_VALID
npm ERR!     at SecurePair.<anonymous> (tls.js:1370:32)
npm ERR!     at SecurePair.EventEmitter.emit (events.js:92:17)
...
```

### ntpdateで時間をあわせる

CERT_NOT_YET_VALIDのエラーは時間がずれていると発生するようです。BeagleBone Blackはバッテリを搭載していないため、電源を入れていないとシステムクロックがずれてしまいます。ntpdateをしてシステムクロックの時間をあわせます。

``` bash
$ sudo ntpdate -b -s -u pool.ntp.org
```

### インストールと起動

インストールし直します。

``` bash
$ npm install orion --production
```

`npm start`でOrionサーバーを起動します。

``` bash
$ npm start orion

> orion@0.0.34 start /home/ubuntu/node_modules/orion
> node server.js

Could not find npm in the following locations:
/usr/bin/../lib/node_modules/npm/bin/npm-cli.js
/usr/bin/./node_modules/npm/bin/npm-cli.js
Please add or modify the npm_path in the orion.conf file and restart the server.

Using workspace: /home/ubuntu/node_modules/orion/.workspace
Listening on port 8081...
```

ブラウザでOrionの起動を確認します。

http://192.168.7.2:8081/

![eclipse-orion-start.png](/2015/02/10/beagleboneblack-eclipse-orion/eclipse-orion-start.png)


### Orionをデモナイズ

一度Orionのプロセスを停止します。pm2にアプリを登録するスクリプトを実行します。

``` bash ~/pm2_orion.sh
#!/bin/bash
read -d '' my_json <<_EOF_
{
  "name"       : "orion",
  "script"     : "/home/ubuntu/node_modules/orion/server.js"
}
_EOF_

echo $my_json | pm2 start -
```

pm2を開始してから、Cloud9を起動します。

``` bash
$ chmod u+x pm2_orion.sh
$ ./pm2_orion.sh
[PM2] Process launched
┌──────────┬────┬──────┬──────┬────────┬───────────┬────────┬────────────┬──────────┐
│ App name │ id │ mode │ PID  │ status │ restarted │ uptime │     memory │ watching │
├──────────┼────┼──────┼──────┼────────┼───────────┼────────┼────────────┼──────────┤
│ orion    │ 1  │ fork │ 1898 │ online │         0 │ 0s     │ 4.117 MB   │ disabled │
└──────────┴────┴──────┴──────┴────────┴───────────┴────────┴────────────┴──────────┘
 Use `pm2 info <id|name>` to get more details about an app
```

OS再起動後もpm2に登録したプロセスが起動するようにinitスクリプトを作成します。

``` bash
$ sudo pm2 startup ubuntu -u ubuntu
$ sudo /etc/init.d/pm2-init.sh restart
```

すでにinitスクリプトを作成している場合は、OS再起動後のためにプロセスをsaveします。

``` bash
$ sudo pm2 save
[PM2] Dumping processes
```

## hello world

### Editor画面

簡単なNode.jsのスクリプトを作成して実行してみます。

Editorから「hello」フォルダを作成します。

* File > New > Folder > hello

helloフォルダを選択し、「world.js」ファイルを作成します。

* New > File > world.js

console.logのコードを書くと警告が表示されました。

![console-is-undefined.png](/2015/02/10/beagleboneblack-eclipse-orion/console-is-undefined.png)


```js world.js
console.log("hello world");
```

OrionはJavaScriptのバリデーションと文法チェックに[ESLint](http://eslint.org/)を採用しています。画面のガイドにしたがって、eslint-envディレクティブを追加します。

```js world.js
/*eslint-env node */
console.log("hello world");
```

### Shell画面

左メニューからShellを選択します。画面下のコンソールから作成したスクリプトを実行します。

``` bash
» node start hello.js
```

![hello-world.png](/2015/02/10/beagleboneblack-eclipse-orion/hello-world.png)


