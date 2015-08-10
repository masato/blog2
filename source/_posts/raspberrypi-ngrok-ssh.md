title: "Raspberry Piをngrokで公開する - Part3: ngrok 2.0でSSHをトンネルする"
date: 2015-08-06 10:50:27
categories:
 - IoT
tags:
 - RaspberryPi
 - ngrok
 - Supervisor
 - SSH
description: ngrokの最新バージョンは2.0.19です。サインアップすると付与されるtokenがすでに2.0になっているため、1.7のクライアントを使ってSSHのトンネルができません。前回はNode.jsのラッパーを使い、ngrokのバージョンは1.7でした。今回は2.0のバイナリをインストールして、ローカルの無線LANにつながっているRaspberry PiにリモートからSSH接続してみます。
---

ngrokの最新バージョンは[2.0.19](https://ngrok.com/whatsnew)です。[サインアップ](https://ngrok.com)すると付与されるtokenがすでに2.0になっているため、1.7のクライアントを使ってSSHのトンネルはできません。[前回](/2015/08/05/raspberrypi-ngrok-slack/)はNode.jsのラッパーを使い、ngrokのバージョンは1.7でした。今回は2.0のバイナリをインストールして、ローカルの無線LANにつながっているRaspberry PiにリモートからSSH接続してみます���

<!-- more -->

## ngrok 2.0.19

Raspberry Piにログインして[ダウンロード](https://ngrok.com/download)ページからLinux/ARMのバイナリをインストールします。

```bash
$ wget https://dl.ngrok.com/ngrok_2.0.19_linux_arm.zip
$ sudo unzip ngrok_2.0.19_linux_arm.zip -d /usr/local/bin
Archive:  ngrok_2.0.19_linux_arm.zip
  inflating: /usr/local/bin/ngrok
$ ngrok version
ngrok version 2.0.19
```

[サインアップ](https://ngrok.com)するとダッシュボードに表示されるtokenを使い、ローカルに設定ファイルを作成します。

```bash
$ ngrok authtoken xxx
Authtoken saved to configuration file: /home/pi/.ngrok2/ngrok.yml
```

### 有料プラン

[pricing](https://ngrok.com/product#pricing)のページによるとFreeプランの場合は同時接続クライアント数が1つです。

* Freeプラン
 * 同時接続できるクライアントは1つ
 * HTTPS/TCP のトンネルのみ利用可能

Raspberry PiのHTTPとSSHを同時に公開することができません。有料プランにすると同時接続クライアント数が増え、独自ドメインが使えるようになります。一番安いIndividualプランは同時接続クライアント数が3つで`$60/年`です。`$5/月`の計算になるので個人の開発環境としてお手頃です。

## 接続テスト

### コマンドからURLを取得

Raspberry Piのシェルでngrokをフォアグラウンドで実行します。`ngrok tcp 22`とするとSSHの22ポートをトンネルすることができます。

```bash
$ ngrok tcp 22
ngrok by @inconshreveable                                       (Ctrl+C to quit)

Tunnel Status                 online
Version                       2.0.19/2.0.19
Web Interface                 http://127.0.0.1:4040
Forwarding                    tcp://0.tcp.ngrok.io:36198 -> localhost:22

Connections                   ttl     opn     rt1     rt5     p50     p90
                              0       0       0.00    0.00    0.00    0.00
```

リモートからSSH接続してみます。標準出力されているForwardingのドメイン名とポートを使います。

```bash
$ ssh pi@0.tcp.ngrok.io -p 36198
```


### ダッシュボードからURLを取得

ngrokの[ダッシュボード](https://dashboard.ngrok.com/)の`Tunnels Online`セクションに現在接続しているクライアントの情報とトンネルのURLが表示されます。

```
#	URL	Client IP
0	tcp://0.tcp.ngrok.io:36198	126.229.158.108
```

## Supervisor

ngrokのSSHトンネルを[Supervisor](http://supervisord.org/)を使いデモナイズします。IoTプロダクション環境向けの[ngrok link](https://ngrok.com/product/ngrok-link)というサービスも開始しているように、ngrokを使うとコネクテッドデバイスの管理が簡単になります。Raspberry Piの電源を入れるとngrokのSSHトンネルが開始するようにしておけば、[ダッシュボード](https://dashboard.ngrok.com/)からURLを確認してリモートからSSH接続ができるようになります。

### 設定

Supervisorの設定ファイルを作成し、`command`にngrokの22ポートのトンネルを指定します。また`--authtoken`フラグでtokenを明示的に指定します。

```bash /etc/supervisor/conf.d/ngrok-ssh.conf
[program:ngrok-ssh]
command=/usr/local/bin/ngrok tcp 22 --authtoken xxx
numprocs=1
redirect_stderr=true
stdout_logfile=/var/log/ngrok.log
user=pi
```

`reread`して作成した設定ファイルをSupervisorに読み込ませます。

```bash
$ sudo supervisorctl reread
ngrok-ssh: available
```


`add`でSupervisorのサブプロセスにngrokを追加します。

```bash
$ sudo supervisorctl add ngrok-ssh
ngrok-ssh: added process group
$ sudo supervisorctl status ngrok-ssh
ngrok                            RUNNING    pid 2504, uptime 0:00:18
```

サブプロセスに追加した後にconfを編集すると、updateが必要なので忘れないようにします。

```bash
$ sudo supervisorctl update ngrok-ssh
```

### テスト

Supervisorからngrokを起動した後も、ngrokコマンドを直接実行した場合と同様にngrokの[ダッシュボード](https://dashboard.ngrok.com/)の`Tunnels Online`セクションにオンラインのクライアント情報が表示されます。

```
#	URL	Client IP
0	tcp://0.tcp.ngrok.io:39016	126.229.158.108
```

このURLとポートを使い、リモートからRaspberry PiにSSH接続します。

```bash
$ ssh pi@0.tcp.ngrok.io -p 39016
```

