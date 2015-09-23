title: "myThingsをはじめよう - Part9: konashiをmyThingsのトリガーとアクションに使う"
date: 2015-09-22 21:24:23
categories:
 - IoT
tags:
 - konashi
 - Framework7
 - myThings
 - RawGit
 - IDCFクラウド
 - フィジカル・コンピューティング
description: 前回はkonashi.jsのアプリをFramework7で書いてHTML5でネイティブ風な画面を作成しました。このコードをForkしてフィジカル・コンピューティングのお試しをiPhoneアプリのmyThingsとkonashi.jsを使って書いてみます。
---

[前回](http://qiita.com/masato/items/cfef50d16173db8aca42)は[konashi.js](http://konashi.ux-xu.com/kjs/)のアプリを[Framework7](http://www.idangero.us/framework7)で書いてHTML5でネイティブ風な画面を作成しました。このコードをForkしてフィジカル・コンピューティングのお試しをiPhoneアプリのmyThingsとkonashi.jsを使って書いてみます。

<!-- more -->

## myThingsとkonashiのフィジカル・コンピューティング

[フィジカル・コンピューティング](https://en.wikipedia.org/wiki/Physical_computing)とはニューヨーク大学 Dan O'Sullivan 教授が提案した、人間の行動や生活環境によりそったコンピュータとの意思疎通の方法を模索する考え方です。コンピュータにセンサーなどの入出力デバイスをつなぎ、人間と情報をやりとりすることで生活を便利にしたり新しいユーザー体験を生み出すことができるようです。

### 参考

* [Hardware Introduction](http://konashi.ux-xu.com/documents/#specs-supportDevice)
* [フィジカル・コンピューティングはここまで来た！ 「スマホで操作できるガジェット」をアプリで自作できるkonashi.jsを知る](http://engineer.typemag.jp/article/konashijs)

[myThings](http://mythings.yahoo.co.jp/)も[konashi](http://konashi.ux-xu.com/)もコネクテッドデバイスをスマホから操作できるという共通した特徴を持っています。この2つをあわせて使ってみるときっと新しいユーザー体験を発見できると思います。

## trigger-1とaction-1の確認

IDCFチャンネルサーバーの仮想マシンにログインして`list`コマンドを実行します。今回使用するtrigger-1とaction-1のuuidとtokenを確認します。

```bash
$ cd ~/iot_apps/meshblu-compose/
$ docker-compose run --rm iotutil list
...
┌───────────┬──────────┬──────────────────────────────────────┐
│ keyword   │ token    │ uuid                                 │
├───────────┼──────────┼──────────────────────────────────────┤
│ trigger-1 │ d74ebedf │ 21c83792-b25e-4ae7-a627-714af57a1a4b │
├───────────┼──────────┼──────────────────────────────────────┤
...
│ action-1  │ 8a781e76 │ 3a78814a-6879-4543-bacf-9a206cd951a6 │
...
```

## jsdo.it

[jsdo.it](http://jsdo.it/)に[myThingsからLチカのアクションとスイッチのトリガー](http://jsdo.it/ma6ato/euTg)という「コード」を作成しました。[前回](/2015/09/15/konashi20-framework7/)作成した[myThingsからLチカ](http://jsdo.it/ma6ato/mDsk)をForkしています。

![konashi-f7-mythings.png](/2015/09/22/mythings-idcfchannel-konashi/konashi-f7-mythings.png)

### HTML

[Framework7](http://www.idangero.us/framework7/#.VgFQXSDtmko)のCSSとJavaScriptを[Rawgit](http://rawgit.com/)からロードします。CDNに公開されていないパッケージをjsdo.itから使う場合に便利です。

[Meshblu](http://meshblu.octoblu.com/)のJavaScript用ライブラリは[こちら](https://cdn.octoblu.com/js/meshblu/latest/meshblu.bundle.js)のCDNから利用できます。NPMの[meshblu-npm](https://github.com/octoblu/meshblu-npm)を[browserify](http://browserify.org/)したものが公開されています。

```html
<!DOCTYPE html>
<html>
    <head>
        <meta charset="utf-8">
        <meta name="viewport" content="width=device-width, initial-scale=1, maximum-scale=1, minimum-scale=1, user-scalable=no, minimal-ui">
        <meta name="apple-mobile-web-app-capable" content="yes">
        <meta name="apple-mobile-web-app-status-bar-style" content="black">

        <!-- Framework7 css -->
        <link rel="stylesheet" href="https://rawgit.com/nolimits4web/Framework7/master/dist/css/framework7.ios.min.css">
        <link rel="stylesheet" href="https://rawgit.com/nolimits4web/Framework7/master/dist/css/framework7.ios.colors.min.css">
        <title>myThings F7</title>
    </head>
    <body>
        <div class="statusbar-overlay"></div>
        <div class="panel-overlay"></div>
        <div class="panel panel-left panel-reveal">
            <div class="content-block">
                <p>Left panel content goes here</p>
            </div>
        </div>
        <div class="views">
            <div class="view view-main">
                <div class="navbar">
                    <div class="navbar-inner">
                        <div class="center sliding">myThings F7</div>
                        <div class="right">
                            <a href="#" class="link icon-only open-panel"><i class="icon icon-bars-blue"></i></a>
                        </div>
                    </div>
                </div>
                <div class="pages navbar-through toolbar-through">
                    <div data-page="index" class="page">
                        <div class="page-content">
                            <div class="content-block">
                                <a href="#" id="btn-find" class="find button button-big">Find konashi</a>
                            </div>
                            <div id="pio-setting">
                                <div class="content-block-title">PIO: Output Settings</div>
                                <div class="list-block">
                                    <ul>
                                        <li class="item-content">
                                            <div class="item-media"><i class="icon icon-form-toggle"></i></div>
                                            <div class="item-inner">
                                                <div class="item-title label">LED2</div>
                                                <div class="item-input">
                                                    <label class="label-switch">
                                                        <div class="toggle" data-pin="1">
                                                            <input type="checkbox">
                                                            <div class="checkbox"></div>
                                                        </div>
                                                    </label>
                                                </div>
                                            </div>
                                        </li>
                                        <li class="item-content">
                                            <div class="item-media"><i class="icon icon-form-toggle"></i></div>
                                            <div class="item-inner">
                                                <div class="item-title label">LED3</div>
                                                <div class="item-input">
                                                    <label class="label-switch">
                                                        <div class="toggle" data-pin="2">
                                                            <input type="checkbox">
                                                            <div class="checkbox"></div>
                                                        </div>
                                                    </label>
                                                </div>
                                            </div>
                                        </li>
                                    </ul>
                                </div>
                                <div class="content-block-title">PIO: Input Settings</div>
                                <div class="list-block">
                                    <ul>
                                        <li class="item-content">
                                            <div class="item-media"><i class="icon icon-form-settings"></i></div>
                                            <div class="item-inner">
                                                <div class="item-title label">S1</div>
                                                <div class="item-after" id="s1-status">OFF</div>
                                            </div>
                                        </li>
                                    </ul>
                                </div>
                            </div>
                        </div>
                    </div>
                </div>
            </div>
        </div>

        <!-- for Framework7 -->
        <script src="https://rawgit.com/nolimits4web/Framework7/master/dist/js/framework7.min.js"></script>
        <!-- meshblu -->
        <script type="text/javascript" src="https://cdn.octoblu.com/js/meshblu/latest/meshblu.bundle.js"></script>
        <!-- for konashijs -->
        <script src="http://konashi.ux-xu.com/kjs/konashi-bridge.min.js"></script>

    </body>
</html>
```

### JavaScript

最初にkonashi.jsからロードするJavaScriptの全文です。Meshbluのデバイスのuuidやtokenは環境に応じて変更します。

```javascript
(function(Framework7, $$){
    $$('.toggle').on("click", function(e){
        var pin = $$(e.currentTarget).data("pin");
        var value = $$(this).find('input').prop('checked') ? k.HIGH : k.LOW;
        k.digitalWrite(pin, value);
    });

    $$("#btn-find").on("click", function(){
        if($$("#btn-find").hasClass("find")){
            k.find();
        } else {
            k.disconnect();

            // change find button
            $$("#btn-find")
              .addClass("find")
              .html("Find konashi");

            // hide pio list
            $$("#pio-setting").hide();
            $$("#s1-status").html("OFF");
        }
    });

    var server = '210.140.162.58',
        port = 80,
        protocol = 'websocket',
        trigger_uuid = '21c83792-b25e-4ae7-a627-714af57a1a4b',
        trigger_token = 'd74ebedf',
        action_uuid = '3a78814a-6879-4543-bacf-9a206cd951a6',
        action_token = '8a781e76';

    var conn;

    k.on("ready", function(){

        // change find button
        $$("#btn-find")
          .removeClass("find")
          .html("Disconnect konashi");

        // show pio list
        $$("#pio-setting").show();

        k.pinModeAll(254);

        conn = meshblu.createConnection({
            'uuid': trigger_uuid,
            'token': trigger_token,
            'server': server,
            'port': port
        });

        conn.on('notReady', function(data){
            k.log('UUID FAILED AUTHENTICATION!');
        });

        conn.on('ready', function(data){
            k.log('READY!!');

            conn.subscribe({
                'uuid': action_uuid,
                'token': action_token
            }, function (data) {
                k.log(data);
            });

            conn.on('message', function(message){
                k.log(message.payload.message);
                var value = message.payload === 'led-on' ? k.HIGH : k.LOW;

                k.digitalWrite(k.LED3, value);
            });
        });
    });

    k.updatePioInput( function(data){
        if(data % 2){
            $$("#s1-status").html("ON");
            conn.data({
                'uuid': trigger_uuid,
                'trigger': 'on'
            });
        } else {
            $$("#s1-status").html("OFF");
        }
    });

    //k.showDebugLog();
})(Framework7, Dom7);
```

`k.on("ready")`のコールバックでkonashiとiPhoneが接続できた後に、IDCFチャンネルサーバーのMeshbluにtrigger-1のuuidを使ってWebSocketで接続します。

```javascript
        conn = meshblu.createConnection({
            'uuid': trigger_uuid,
            'token': trigger_token,
            'server': server,
            'port': port
        });
```

Meshbluに接続すると次にaction-1のuuidでWebSocketのsubscribeをします。ここではmyThingsのトリガーから「led-on」のメッセージを受信した場合にLED3を点灯し、それ以外で消灯します。

```javascript
        conn.on('ready', function(data){
            k.log('READY!!');

            conn.subscribe({
                'uuid': action_uuid,
                'token': action_token
            }, function (data) {
                k.log(data);
            });

            conn.on('message', function(message){
                k.log(message.payload.message);
                var value = message.payload === 'led-on' ? k.HIGH : k.LOW;

                k.digitalWrite(k.LED3, value);
            });
        });
```

konashiのタクトスイッチを押すと、WebSocketクライアントから`data`のAPIに任意のデータを送信します。myThingsの組合せに設定した「IDCF」チャンネルの`trigger-1`のトリガーが発火され、同様に組合せに設定されたアクションが実行されます。

```javascript
    k.updatePioInput( function(data){
        if(data % 2){
            $$("#s1-status").html("ON");
            conn.data({
                'uuid': trigger_uuid,
                'trigger': 'on'
            });
        } else {
            $$("#s1-status").html("OFF");
        }
    });
```

### CSS

スタイルシートはkonaashiがiPhoneと接続、切断に応じて画面のコントロール表示を切り替えます。

```css
#pio-setting {
    display: none;
}
```


## konashiをmyThingsのトリガーに使う

konashiのタクトスイッチを押す(ON)と、myThingsの組合せでトリガーに選択した「IDCF」チャンネルのtrigger-1が発火され、アクションに設定した「Twitter」チャンネルを実行するサンプルです。

### 組合せの作成

myThingsアプリでトリガーにIDCF、アクションにTwitterを選択して組み合わせを作成します。

![konashi-idcf-recipe1.png](/2015/09/22/mythings-idcfchannel-konashi/konashi-idcf-recipe1.png)

### IDCFのトリガー

「IDCF」チャンネルのトリガーはtrigger-1を選択します。

![konashi-idcf-trigger.png](/2015/09/22/mythings-idcfchannel-konashi/konashi-idcf-trigger.png)

trigger-1はkonashi.jsのJavaScriptからMeshbluへの接続に使っています。

```js
        conn = meshblu.createConnection({
            'uuid': trigger_uuid,
            'token': trigger_token,
            'server': server,
            'port': port
        });
```

### Twitterのアクション

ツイート内容は任意のメッセージを登録します。

![konashi-twitter-action.png](/2015/09/22/mythings-idcfchannel-konashi/konashi-twitter-action.png)


### 手動実行

konashi.jsはバックグラウンドで実行できないので、myThingsアプリの「手動実行」ボタンと同時に使うことはできません。15分待つか別のスマホからmyThingsアプリを起動して使います。

konashiのタクトスイッチを押したあと、組み合わせの「手動実行」ボタンを押すとアクションに設定してあるメッセージがツイートされます。

![konashi-tweet.png](/2015/09/22/mythings-idcfchannel-konashi/konashi-tweet.png)

## konashiをmyThingsのアクションに使う

myThingsの組合せでトリガーの「Gmail」で設定するように、Gmailが特定の誰かからメールを受信すると、アクションに選択した「IDCF」チャンネルのaction-1が実行されます。メールの件名の応じてkonashiのLED3が点灯と消灯をするサンプルです。


### 組合せの作成

myThingsアプリでトリガーにGmail、アクションにIDCFを選択して組み合わせを作成します。

![konashi-idcf-recipe2.png](/2015/09/22/mythings-idcfchannel-konashi/konashi-idcf-recipe2.png)


### Gmailのトリガー

Gmailのトリガー条件は「特定の誰かからメールを受信したら」を選択します。トリガーとして使うメールアドレスを登録します。

![konashi-idcf-gmail.png](/2015/09/22/mythings-idcfchannel-konashi/konashi-idcf-gmail.png)

### IDCFのアクション

IDCFのアクションはaction-1を選択します。メッセージは「候補選択」からGmailの{{件名}}を設定します。

![konashi-idcf-action.png](/2015/09/22/mythings-idcfchannel-konashi/konashi-idcf-action.png)

### 手動実行

トリガー条件に設定したメールアドレスからmyThingsと連携したGmailのメールアドレスにメールを送信します。件名を「led-on」とするとLED3が点灯します。それ以外の場合はLED3が消灯します。

![konashi-led-on.png](/2015/09/22/mythings-idcfchannel-konashi/konashi-led-on.png)

Gmailでメールの受信を確認したら、作成した組合せの「手動実行」ボタンを押すとLED3が点灯または消灯します。
