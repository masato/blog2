title: 'BeagleBone BlackにCloud9をインストール'
date: 2014-05-04 03:39:44
tags:
 - Ubuntu
 - BeagleBoneBlack
 - Cloud9
 - pm2
description: 前回、Ubuntuのインストールが終わったので、Cloud9を構築してみます。Cloud9は、c9.ioというCloudIDEのサービスがありますが、オープンソース版でも、Node.jsだけなら開発できます。
---

* Update: [BeagleBone BlackのUbuntu14.04.1にNode.jsとnpmをインストールする](/2015/02/09/beagleboneblack-nodejs-npm/)

[前回](/2014/05/03/beagleboneblack-ubuntu)、Ubuntuのインストールが終わったので、[Cloud9](http://www.cloud9ide.com/)を構築してみます。
Cloud9は、[c9.io](https://c9.io/)というCloudIDEのサービスがありますが、[オープンソース](https://github.com/ajaxorg/cloud9)版でも、Node.jsだけなら開発できます。

<!-- more -->

### Cloud9のインストール

`BeagleBone Black`をWindows7のノートPCとUSB接続をして、SSHでログインします。
Cloud9自体もNode.jsでできているので、UbuntuにARM用のNode.jsをインストールします。Cloud9はGitHubからclone後に、npm installします。

``` bash
$ sudo apt-get update
$ sudo apt-get install xz-utils git libxml2-dev libxml2-dev make mercurial
$ wget http://s3.armhf.com/debian/saucy/node-v0.10.21-saucy-armhf.tar.xz
$ sudo tar xJvf node-v*-saucy-armhf.tar.xz -C /usr/local --strip-components 1
$ git clone https://github.com/ajaxorg/cloud9.git
$ cd cloud9
$ npm install
```

### pm2を使いデーモン化

[pm2](https://github.com/Unitech/pm2)を使い、Cloud9が自動起動するようにします。

pm2のインストール

``` bash
$ sudo apt-get update
$ sudo apt-get install libcap2-bin gcc g++ libssl-dev
$ sudo setcap cap_net_bind_service=+ep /usr/local/bin/node
$ sudo npm install -g pm2
```

pm2の起動スクリプト

``` bash ~/pm2_cloud9.sh
#!/bin/bash
read -d '' my_json <<_EOF_
{
  "name"       : "cloud9",
  "script"     : "./bin/cloud9.sh",
  "cwd"        : "/home/ubuntu/cloud9",
  "args"      : ["-l","0.0.0.0","-w","/home/ubuntu/applications"]
}
_EOF_

echo $my_json | pm2 start -
```

pm2を開始してから、Cloud9を起動します。

``` bash
$ chmod u+x pm2_cloud9.sh
$ ./pm2_cloud9.sh
$ touch /home/ubuntu/.pm2/dump.pm2
$ sudo pm2 startup ubuntu -u ubuntu
$ sudo /etc/init.d/pm2-init.sh restart
```

USB接続したIPアドレスの3131ポートに、Windows7のブラウザから確認します。

```
http://[BeagleBone Blackのusb0]:3131
```

### まとめ

ローカルにポータブルなNode.jsの開発環境ができました。
BeagleBone BlackをUSBポートに接続したWindowsやOSXから、シリアル接続でCloud9を使うことができます。
実際にはこの環境でNode.jsアプリを開発することはありませんが、ローカルのCloudIDEというアイデア実証としてはおもしろいです。
