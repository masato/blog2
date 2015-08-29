title: "myThingsをはじめよう - Part2: IDCFクラウドに「IDCF」チャンネルをセットアップする"
date: 2015-08-25 09:43:38
categories:
 - IoT
tags:
 - myThings
 - IDCFクラウド
 - IDCFチャンネル
 - Meshblu
---


[myThings](http://mythings.yahoo.co.jp/)に自分のデバイスを接続することができる[「IDCF」チャンネル](https://github.com/IDCFChannel/meshblu-compose)は、[Meshblu](https://github.com/octoblu/meshblu/) IoT Platform を採用しています。また[Docker Compose](https://github.com/docker/compose)を使い複数のサービスを構成管理しています。はじめてIDCFクラウドを使う方を対象に「IDCF」チャンネル用の仮想マシンをセットアップする手順をまとめます。

<!-- more -->

## IDCFクラウドに仮想マシンを作成する

WebブラウザからIDCFクラウドの[クラウドコンソール](https://idcfcloud.com/?cl=cl_gnv)にログインして作業します。左上の「仮想マシンを作成」ボタンをクリックします。

### マシンタイプとイメージ

最初にUbuntu 14.04の仮想マシンを新規作成します。


* マシンタイプ: light.S1
* イメージ: Ubuntu Server 14.04 LTS 64-bit

![idcfcloud-type-image.png](/2015/08/25/mythings-idcfchannel-setup/idcfcloud-type-image.png)


### SSH Key

SSH Keyは「おすすめTemplate」から選択したイメージはSSHのパスワードログインができません。まだSSH Keyを登録していない場合は、クラウドコンソール上で作成、またはローカルにあるRSA公開鍵をアップロードしてから選択します。


* ボリューム: デフォルト (なし)
* SSH Key: 選択
* 仮想マシン台数: デフォルト (1)
* ネットワークインタフェース: デフォルト

![idcfcloud-ssh.png](/2015/08/25/mythings-idcfchannel-setup/idcfcloud-ssh.png)


### 詳細情報

マシン名1に任意の名前を付けたあと、「確認画面へ」ボタンをクリックします。

* 詳細情報 -> マシン名1: 任意の名前

![idcfcloud-hostname.png](/2015/08/25/mythings-idcfchannel-setup/idcfcloud-hostname.png)


### 仮想マシン作成確認

最後に作成する仮想マシンの情報を確認します。間違いが無ければ「作成」ボタンをクリックして仮想マシンを作成します。

![idcfcloud-create.png](/2015/08/25/mythings-idcfchannel-setup/idcfcloud-create.png)


仮想マシンの一覧画面に戻り、しばらくすると作成した仮想マシンのステータスが「Running」になります。

![idcfcloud-vmlist.png](/2015/08/25/mythings-idcfchannel-setup/idcfcloud-vmlist.png)

### IPアドレス

メニューから「IPアドレス」を選択します。はじめての場合は「NAT」列が「ソース」と表示されたIPアドレスが1つ表示されています。「IPアドレス名」をクリックして設定画面を開きます。

### IPアドレス > ファイアウォール

「ファイアウォール」は4つ登録します。HTTPSはmyThingsサーバーから接続するため「ソースCIDR」は「Any(0.0.0.0/0)」を選択します。

HTTP、MQTTはデバイスが「IDCF」チャンネルに接続できるように設定します。今回は2つともすべて「Any(0.0.0.0/0)」にしました。

SSHは作業マシンから「IDCF」チャンネルの仮想マシンにSSH接続できるように設定します。ここでは「My IP」を選択して、現在「クラウドコンソール」を開いている作業マシンの「ソースCIDR」を選択します。

* HTTP: 80 port, ANY
* HTTPS: 443 port, ANY
* MQTT: 1883 port, ANY
* SSH: 22 port, SSHクライアント


![idcfcloud-firewall.png](/2015/08/25/mythings-idcfchannel-setup/idcfcloud-firewall.png)


### IPアドレス > ポートフォワード

「ポートフォワード」も同様に4つ登録します。先ほど作成した仮想マシン名を選択して登録していきます。

* HTTP: 80 port
* HTTPS: 443 port
* MQTT: 1883 port
* SSH: 22 port

![idcfcloud-portforward.png](/2015/08/25/mythings-idcfchannel-setup/idcfcloud-portforward.png)

以上でクラウドコンソール画面から仮想マシンの作成とIPアドレスの設定作業は終了です。

### SSH接続する

「IPアドレス > ファイアウォール」画面でSSHの接続を登録した作業マシンからSSH接続します。ユーザーは「root」、クラウドコンソール画面で選択した秘密鍵を指定します。

```bash
$ ssh root@210.140.162.58 -i ~/.ssh/id_rsa
root@mythings:~#
```

## 「IDCF」チャンネルをインストールする

### シェルスクリプトの実行

「IDCF」チャンネルはGitHubのリポジトリからcloneしてインストールします。最初にGitをインストールします。

```bash
$ sudo apt-get update && sudo apt-get install -y git
```

適当なディレクトリを作成して、「IDCF」チャンネルの[リポジトリ](https://github.com/IDCFChannel/meshblu-compose)から`git clone`します。

```bash
$ mkdir ~/iot_apps && cd ~/iot_apps
$ git clone https://github.com/IDCFChannel/meshblu-compose
```

cloneしたディレクトリに移動してbootstrapのシェルスクリプトを実行してインストールします。

```bash
$ cd meshblu-compose
$ ./bootstrap.sh
```

### インストールの確認

「IDCF」チャンネルはMeshbluとデータベース、コマンドラインツールなどのDockerイメージを使い、Docker Composeによって構成されています。インストールが成功すると、`docker-compose ps`コマンドの結果が表示されます。コンテナが4つ起動しました。

```bash
$ docker-compose ps
      Name             Command             State              Ports
-------------------------------------------------------------------------
meshblucompose_m   npm start          Up                 0.0.0.0:1883->18
eshblu_1                                                 83/tcp, 80/tcp
meshblucompose_m   /entrypoint.sh     Up                 27017/tcp
ongo_1             mongod
meshblucompose_o   nginx -c /etc/ng   Up                 0.0.0.0:443->443
penresty_1         inx/nginx.conf                        /tcp, 0.0.0.0:80
                                                         ->80/tcp
meshblucompose_r   /entrypoint.sh     Up                 6379/tcp
edis_1             redis-server
```

curlコマンドを実行してMesubluサーバーが起動していることを確認します。

```bash
$ curl https://localhost/status
{"meshblu":"online"}
```

### サービスの起動と再起動

停止状態からサービスを起動するときはOpenRestyサービスを`up`コマンドから起動します。

```bash
$ docker-compose up -d openresty
```

サービスを再起動するときは`restart`コマンドを実行します。

```bash
$ docker-compose restart
```

## デバイスの準備

### 「デバイス」について

MeshbluではHTTP、MQTT、WebSoketを使いメッセージを送受信するクライアントを「デバイス」として管理します。Raspberry PiやWebサービス、コマンドなどがこの「デバイス」に該当します。


### デバイスの登録

インストール直後はデバイスは登録されていません。最初にdocker-compose.ymlに付属しているコマンドラインツールを使いデバイスを登録します。

```bash
$ cd ~/iot_apps/meshblu-compose
$ docker-compose run --rm iotutil register

> iotutil@0.0.1 start /app
> node app.js "register"

┌─────────┬────────────────────┬──────────────────────────────────────┐
│ keyword │ meshblu_auth_token │ meshblu_auth_uuid                    │
├─────────┼────────────────────┼──────────────────────────────────────┤
│ owner   │ 12749000           │ 847cda3d-7999-4b95-b8ec-af07652cb842 │
└─────────┴────────────────────┴──────────────────────────────────────┘
```

コマンドを実行すると、`trigger-[1-5]`と`action-[1-5]`とownerをあわせて合計11個のデバイスが登録されます。標準出力されるownerのデバイスは作成した11個のデバイスすべてにメッセージを送信することができます。デフォルトでは、`trigger-[1-5]`から`action-[1-5]`へ末尾が同じ数字の組み合わせのメッセージ送信を許可しています。

### デバイス情報の取得

登録したデバイスは`list`コマンドで確認することができます。

```bash
$ docker-compose run --rm iotutil list

> iotutil@0.0.1 start /app
> node app.js "list"

┌───────────┬──────────┬──────────────────────────────────────┐
│ keyword   │ token    │ uuid                                 │
├───────────┼──────────┼──────────────────────────────────────┤
│ trigger-1 │ d286ba8d │ ffa6934d-f1b3-467f-98b3-766b330d436d │
├───────────┼──────────┼──────────────────────────────────────┤
│ trigger-2 │ 77eecd8e │ 5ebfd5bb-4f51-45a7-a66d-12037352d0c7 │
├───────────┼──────────┼──────────────────────────────────────┤
│ trigger-3 │ 933f5a29 │ f1c63bf4-905f-4065-afbe-ac152fa76806 │
├───────────┼──────────┼──────────────────────────────────────┤
│ trigger-4 │ bc4510e7 │ 75498ace-32eb-4b94-971c-5812db48633d │
├───────────┼──────────┼──────────────────────────────────────┤
│ trigger-5 │ 0d0ba029 │ 4c6a9051-790a-4f3d-8cdf-2b8faf34e21c │
├───────────┼──────────┼──────────────────────────────────────┤
│ action-1  │ 8a83d71f │ b61d3398-ac99-4694-9dc4-dd632faf6f6a │
├───────────┼──────────┼──────────────────────────────────────┤
│ action-2  │ 042bef69 │ 16d59d34-a4e0-482d-a96f-ecba3a3ac87a │
├───────────┼──────────┼──────────────────────────────────────┤
│ action-3  │ ec4e5652 │ 9824ee09-7f0f-4fdb-b0e7-13146272056b │
├───────────┼──────────┼──────────────────────────────────────┤
│ action-4  │ db92042d │ eea855c7-0a06-4789-b0b1-4a0f661c34b3 │
├───────────┼──────────┼──────────────────────────────────────┤
│ action-5  │ 0338410c │ e40d6036-a326-4b3c-bffc-bc91b1365470 │
└───────────┴──────────┴──────────────────────────────────────┘
```

`show`コマンドを実行すると個別のデバイスの情報を表示します。以下は`--keyword`フラグに`action-1`を指定した例です。

```bash
$ docker-compose run --rm iotutil show -- --keyword action-1

> iotutil@0.0.1 start /app
> node app.js "show" "--keyword" "action-3"

┌──────────┬──────────┬──────────────────────────────────────┐
│ keyword  │ token    │ uuid                                 │
├──────────┼──────────┼──────────────────────────────────────┤
│ action-1 │ b6716f76 │ 32d85553-b41d-4a62-8828-b5f1187768ee │
└──────────┴──────────┴──────────────────────────────────────┘
```

`owner`コマンドを実行するとownerのデバイス情報を表示します。

```bash
$ docker-compose run --rm iotutil owner

> iotutil@0.0.1 start /app
> node app.js "owner"

┌─────────┬────────────────────┬──────────────────────────────────────┐
│ keyword │ meshblu_auth_token │ meshblu_auth_uuid                    │
├─────────┼────────────────────┼──────────────────────────────────────┤
│ owner   │ 12749000           │ 847cda3d-7999-4b95-b8ec-af07652cb842 │
└─────────┴────────────────────┴──────────────────────────────────────┘
```

### デバイス間のメッセージを許可する

末尾の番号が同じ番号の場合は、`trigger-*`から`action-*`へメッセージが送信できるように設定されています。

* 例: triggger-1からaction-1

任意のデバイス間のメッセージ送信を許可したいときは、以下のコマンドを実行します。action-1からtrigger-3へのメッセージ送信を許可します。

```bash
$ docker-compose run --rm iotutil whiten -- -f action-1 -t trigger-3

> iotutil@0.0.1 start /app
> node app.js "whiten" "-f" "action-1" "-t" "trigger-3"

action-1 can send message to trigger-3
```


## デバイスの削除

`del`コマンドを実行すると登録してあるデバイスをすべて削除できます。削除後は`regiser`コマンドを実行してデバイスを再作成します。

```bash
$ docker-compose run --rm iotutil del

> iotutil@0.0.1 start /app
> node app.js "del"

are you sure?: [Yn]:  Y
trigger-1, trigger-2, trigger-3, trigger-4, trigger-5, action-1, action-2, action-3, action-4, action-5, owner are deleted.
```