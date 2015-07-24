title: "Part3 myThingsからIDCFチャンネルを使う"
date: 2015-07-22 10:23:13
tags:
---

## Part3 myThingsからIDCFチャンネルを使う

最初にmyThingsからIDCFチャンネルを使うために必要なIPアドレスと認証tokenを確認します。

### IDCFチャンネルのIPアドレス

IDCFチャンネルのIPアドレスを確認します。ブラウザからIDCFクラウドにログインします。クラウドコンソールの「IPアドレス」をクリックしてIDCFチャンネルが使っているIPアドレスを確認します。新規アカウントを作成するとIPアドレスが1つ作成されています。複数のIPアドレスがある場合は、IDCFチャンネルの仮想マシンにポートフォワードしているIPアドレスを使います。

### IDCFチャンネルの認証token

次に仮想マシンにログインしてIDCFチャンネルが起動しているディレクトリに移動します。

```bash
$ cd ~/iot_apps/meshblu-compose
```

以下のコマンドを実行して認証tokenを確認します。この例では`meshblu_auth_token`列の`b8c55ad8`が認証tokenになります。

```bash
$ docker-compose run --rm iotutil owner

> iotutil@0.0.1 start /app
> node app.js "owner"

┌─────────┬────────────────────┬──────────────────────────────────────┐
│ keyword │ meshblu_auth_token │ meshblu_auth_uuid                    │
├─────────┼────────────────────┼──────────────────────────────────────┤
│ owner   │ b8c55ad8           │ 3928f881-df17-42df-9c59-e0f13334cf26 │
└─────────┴────────────────────┴──────────────────────────────────────┘
```

### IDCFチャンネルを認証する

myThingsを起動してチャンネル一覧を表示します。「開発者向け」にある「IDCF」をタップします。「認証する」をタップしてIDCFチャンネルの認証を行います。次にIPアドレスと認証tokenを入力する画面が表示されます。上記で確認したIPアドレスと認証tokenを入力し、「利用を開始する」をタップします。認証に成功すると「IDCFは認証済みです」と表示されます。

### IDCFチャンネルとSlackの組み合わせを作成する

右上の「プラス」をタップして組み合わせを作成します。「トリガー」をタップして「IDCFチャンネル」を選択します。「条件を満たしたら」をタップします。trigger-1からtrigger-5まで選べますが、今回は「trigger-1」を選択し「OK」をタップします。次に「アクション」をタップして「Slack」を承認します。チャットチャンネルの選択と、IDCFチャンネルのトリガーが「条件を満たした」ときにSlackに表示されるメッセージを入力します。「OK」をタップしてアクション画面を終了します。最初の組み合わせ作成画面に戻り、「作成」をタップすると組み合わせの作成は終了します。

### 組み合わせをテストする

IDCFチャンネルのトリガーが「条件を満たした」状態をテストします。IDCFチャンネルの仮想マシンにログインして、以下のコマンドを実行します。「trigger-1」のuuidとtokenを確認します。

```bash
$ docker-compose run --rm iotutil show -- --keyword trigger-1

> iotutil@0.0.1 start /app
> node app.js "show" "--keyword" "trigger-1"

┌───────────┬──────────┬──────────────────────────────────────┐
│ keyword   │ token    │ uuid                                 │
├───────────┼──────────┼──────────────────────────────────────┤
│ trigger-1 │ d4ab4657 │ 6d6fe678-5ba9-48b1-a2ad-6ca4d540700a │
└───────────┴──────────┴──────────────────────────────────────┘
```

このuuidとtokenを認証用のHTTPヘッダに使います。以下のように`{idcf channel url}/data/{uuid}`のURLにデータをPOSTします。REST APIの場合はHTTPS通信をします。自己署名の証明書を使っているため、curlコマンドの場合は`--insecure`フラグが必要です。

```bash
$ curl -X POST \
  "https://210-140-160-194.jp-east.compute.idcfcloud.com/data/6d6fe678-5ba9-48b1-a2ad-6ca4d540700a" \
  --insecure \
  -d 'led=on' \
  --header "meshblu_auth_uuid: 6d6fe678-5ba9-48b1-a2ad-6ca4d540700a" \
  --header "meshblu_auth_token: d4ab4657"
```

MQTTの場合はトピック名は`data`で固定です。メッセージ送信先uuidはJSON形式のメッセージの`devices`セクションに指定します。MQTTブローカーの認証用のユーザー名とパスワードは、それぞれ上記で確認したuuidとtokenを指定します。

```bash
$ mosquitto_pub \
  -h 210-140-160-194.jp-east.compute.idcfcloud.com  \
  -p 1883 \
  -t data \
  -m '{"devices": ["6d6fe678-5ba9-48b1-a2ad-6ca4d540700a"], "payload": "led on"}' \
  -u 6d6fe678-5ba9-48b1-a2ad-6ca4d540700a \
  -P d4ab4657 \
```

REST APIの`/data/{uuid}`やMQTTの`data`トピックはmyThingsのトリガーを発火するための専用の方法です。通常のMQTTブローカーとしてのpubsubとは異なりますのでご注意ください。

最後にmyThingsの「作成組み合わせ一覧」から組み合わせを選び「手動実行」をタップします。IDCFチャンネルのトリガーが発火され、Slackに指定したメッセージが表示されます。

