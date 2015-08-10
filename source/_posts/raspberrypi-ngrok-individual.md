title: "Raspberry Piをngrokで公開する - Part4: 有料プランでサブドメインとTCPアドレスを予約する"
date: 2015-08-07 10:50:27
categories:
 - IoT
tags:
 - RaspberryPi
 - ngrok
 - Supervisor
 - SSH
description: 前回試したのはngrokのFreeプランだったので、2つのクライアントを同時に起動できませんでした。Raspberry Piで起動している複数のサービスのポートをインターネット上に公開したいのでIndividualの有料プランを契約することにします。
---

[前回](/2015/08/06/raspberrypi-ngrok-ssh/)試したのはngrokのFreeプランだったので、2つのクライアントを同時に起動できませんでした。Raspberry Piで起動している複数のサービスのポートをインターネット上に公開したいのでIndividualの[有料プラン](https://ngrok.com/product#pricing)を契約することにします。


<!-- more -->

## プランの変更

あらかじめ[ダッシュボード](https://dashboard.ngrok.com/)に表示されるtokenを使いローカルに設定ファイルを作成しておきます。

```bash
$ ngrok authtoken xxx
Authtoken saved to configuration file: /home/pi/.ngrok2/ngrok.yml
```

ngrokのダッシュボードに[ログイン](https://dashboard.ngrok.com/user/login)します。[Billing](https://dashboard.ngrok.com/billing/)タブのPlanセクションに表示される`Change Plan`ボタンをクリックします。Individualプランは年契約で$60を一括で支払います。


![ngrok-individual.png](/2015/08/07/raspberrypi-ngrok-individual/ngrok-individual.png)


## サブドメインの予約

独自ドメインも使えますが、[DNS](https://ngrok.com/docs#custom-domains)の変更が必要になります。開発環境なので`ngrok.io`のサブドメインを使うことにします。通常はサブドメインはランダムで自動的に割り当てられますが、有料プランに変更すると自分で好きな名前を付けることができます。トンネルがオフラインでもサブドメインは確保されます。

Individualプランの場合は3つまでサブドメイン名を予約しておくことが出来ます。[Reserved Tunnels](https://dashboard.ngrok.com/reserved)画面のReserved DomainセクションにあるNameテキストボックスに予約したいアプリ名などの名前を入力して、`Reserve Domain`ボタンをクリックします。


![reserved-domain.png](/2015/08/07/raspberrypi-ngrok-individual/reserved-domain.png)


### 使い方

[Docs](https://ngrok.com/docs)の[Custom subdomain names](https://ngrok.com/docs#subdomain)セクションに使い方があります。

```bash
$ ngrok http -subdomain=inconshreveable 80
```

HTTPトンネルはSupervisorのサブプロセスとして起動しているので最初に停止しておきます。

```bash
$ sudo supervisorctl stop ngrok-http
```

設定ファイルの`command`行にあるngrokコマンドに`--authtoken`フラグとあわせて、`-subdomain`フラグに予約したサブドメイン名を追加します。

```bash /etc/supervisor/conf.d/ngrok-http.conf
[program:ngrok-http]
command=/usr/local/bin/ngrok http --authtoken xxx -subdomain=xxx 3000
numprocs=1
redirect_stderr=true
stdout_logfile=/var/log/ngrok-http.log
user=pi
```

Supervisorの設定ファイルを編集したのでupdateして再起動します。

```bash
$ sudo supervisorctl update ngrok-http
ngrok-http: stopped
ngrok-http: updated process group
```

ダッシュボードのTunnels Onlineセクションに予約されたドメイン名と接続しているクライアントのIPアドレスが表示されます。

![reserved-domain-assign.png](/2015/08/07/raspberrypi-ngrok-individual/reserved-domain-assign.png)

リモートホストからcurlでURLを開いてトンネルを確認します。

```bash
$ curl http://xxx.ngrok.io
Hello World!
```

## TCPアドレスの予約

SSHのトンネルを使うときは以下のようにTCPアドレスを使います。通常はポート番号はクライアントからトンネルを作成するごとランダムになります。

```bash
$ ssh pi@0.tcp.ngrok.io -p 36198
```

Individualプランの場合は2つまで予約することができます。[Reserved Tunnels](https://dashboard.ngrok.com/reserved)画面の`Reserve Address`ボタンをクリックするとポート番号を予約することができます。


![reserved-address.png](/2015/08/07/raspberrypi-ngrok-individual/reserved-address.png)


### 使い方

[Docs](https://ngrok.com/docs)の[Listening on a reserved remote address](https://ngrok.com/docs#remote-addr)セクションに使い方があります。

```bash
$ ngrok tcp --remote-addr 1.tcp.ngrok.io:20301 22
```

最初にSupervisorのサブプロセスを停止します。

```bash
$ sudo supervisorctl stop ngrok-ssh
```

設定ファイルの`command`行にあるngrokコマンドに`--authtoken`フラグとあわせて、`--remote-addr`フラグに予約したTCPアドレスを追加します。

```bash /etc/supervisor/conf.d/ngrok-ssh.conf
[program:ngrok-ssh]
command=/usr/local/bin/ngrok tcp --authtoken xxx --remote-addr 1.tcp.ngrok.io:xxx 22
numprocs=1
redirect_stderr=true
stdout_logfile=/var/log/ngrok-ssh.log
user=pi
```

Supervisorの設定ファイルを編集したのでupdateして再起動します。

```bash
$ sudo supervisorctl update ngrok-ssh
ngrok-ssh: stopped
ngrok-ssh: updated process group
```

ダッシュボードのTunnels Onlineセクションに予約されたTCPアドレスと、接続しているクライアントのIPアドレスが表示されます。


![reserved-address-assign.png](/2015/08/07/raspberrypi-ngrok-individual/reserved-address-assign.png)


このTCPアドレスを使ってローカルネットワークにあるRaspberry PiにリモートからSSH接続ができるようになります。

```bash
$ ssh pi@1.tcp.ngrok.io -p xxx
```

SSH接続の確認ができたら一度ログアウトします。 ssh-copy-idコマンドなどを使いSSHクライアントの公開鍵をコピーします。

```bash
$ ssh-copy-id -i ~/.ssh/id_rsa.pub pi@1.tcp.ngrok.io -p xxx
```

リモートからSSH接続をするのでsshdの設定を変更します。最初にバックアップをします。

```bash
$ sudo cp /etc/ssh/sshd_config{,.orig}
```

ルートログイン不可、パスワード認証不可にしておきます。

```bash /etc/ssh/sshd_config
PermitRootLogin no
PasswordAuthentication no
PermitEmptyPasswords no
```

最後に設定ファイルをreloadします。

```bash
$ sudo service ssh reload
```
