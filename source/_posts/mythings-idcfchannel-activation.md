title: "myThingsをはじめよう - Part4: 「IDCF」チャンネルを認証する"
date: 2015-08-30 11:38:21
categories:
 - IoT
tags:
 - IoT
 - Meshblu
 - myThings
 - IDCFクラウド
description: 前回は「IDCF」チャンネルを独立したMQTTブローカーとして使う例を書きました。準備ができたのでmyThingsのチャンネルでIDCFを使う設定に入っていこうと思います。
---

[前回](/2015/08/26/mythings-idcfchannel-messaging/)は「IDCF」チャンネルを独立したMQTTブローカーとして使う例を書きました。準備ができたので[myThings](http://mythings.yahoo.co.jp/)のチャンネルでIDCFを使う設定に入っていこうと思います。

## 「IDCF」チャンネルの認証情報

myThingsアプリから「IDCF」を認証する場合、以下の2つの情報が必要になります。

* IPアドレス: IDCFクラウドの仮想マシンのパブリックIPアドレス
* 認証token: 「IDCF」チャンネルのownerのtoken

### パブリックIPアドレス

IPアドレスは[IDCFクラウドに「IDCF」チャンネルをセットアップ](/2015/08/25/mythings-idcfchannel-setup/)したとき、仮想マシンに割り当てたパブリックのIPアドレスを使います。この例ではインターネットからアクセス可能な`210.140.162.58`です。

![idcfcloud-firewall.png](/2015/08/30/mythings-idcfchannel-activation/idcfcloud-firewall.png)

### ownerのtoken

認証tokenは「IDCF」チャンネルを仮想マシンにインストールしたあと、最初に実行する`register`コマンドで表示されますが、あとから確認することもできます。インストールしたディレクトリに移動後`owner`コマンドを実行します。以下の場合は画面に出力された`meshblu_auth_token`の`18633a7c`を使います。

```bash
$ docker-compose run --rm iotutil owner

> iotutil@0.0.1 start /app
> node app.js "owner"

┌─────────┬────────────────────┬──────────────────────────────────────┐
│ keyword │ meshblu_auth_token │ meshblu_auth_uuid                    │
├─────────┼────────────────────┼──────────────────────────────────────┤
│ owner   │ 18633a7c           │ 12a95246-f31e-4f0c-9678-2ae755e57b98 │
└─────────┴────────────────────┴──────────────────────────────────────┘
```

## 「IDCF」チャンネルを認証する

### IDCFを選択する

myThingsアプリを開いてチャンネル一覧の一番下にある「開発向け」セクションから「IDCF」を選択します。

![idcf-channel-list.png](/2015/08/30/mythings-idcfchannel-activation/idcf-channel-list.png)

「認証する」ボタンをタップするとIPアドレスと認証tokenを入力する画面が表示されます。

![idcf-channel.png](/2015/08/30/mythings-idcfchannel-activation/idcf-channel.png)

### 利用を開始する

確認したIPアドレスと認証tokenをmyThingsアプリに入力して、「利用を開始する」ボタンをタップします。

![idcf-channel-token.png](/2015/08/30/mythings-idcfchannel-activation/idcf-channel-token.png)

### 認証の確認

「IDCFは認証済みです」と表示されると、チャンネルの認証に成功です。

![idcf-channel-activated.png](/2015/08/30/mythings-idcfchannel-activation/idcf-channel-activated.png)

チャンネル一覧画面を表示すると、IDCFにチェックマークが入るのでここからも認証済を確認することができます。

![idcf-channel-list-activated.png](/2015/08/30/mythings-idcfchannel-activation/idcf-channel-list-activated.png)
