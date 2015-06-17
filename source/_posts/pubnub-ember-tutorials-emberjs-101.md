title: "PubNubとEmber.jsのチュートリアル - Part1: Ember.js 101"
date: 2015-01-26 23:10:43
tags:
 - PubNub
 - Emberjs
 - Bower
 - Docker
description: PubNubはConnected Devices用のリアルタイムアプリを構築するためBaaSです。リアルタイムメッセージに強いので、FirebaseやRealtime.co、SyncanoといったBaaSに近い感じです。今回はWebブラウザをクライアントにして、Ember.jsのSDKを使ったチュートリアルを進めていきます。AngularJSよりEmber.jsの方が昔から好きなのでコードを書いていて楽しいです。
---

[PubNub](http://www.pubnub.com/)はConnected Devices用のリアルタイムアプリを構築するためBaaSです。リアルタイムメッセージに強いので、[Firebase](http://www.firebase.com/)や[Realtime.co](http://www.realtime.co/)、[Syncano](http://www.syncano.com/)といったBaaSに近い感じです。今回はWebブラウザをクライアントにして、[Ember.jsのSDK](https://github.com/pubnub/pubnub-ember/)を使ったチュートリアルを進めていきます。AngularJSより[Ember.js](http://emberjs.com/)の方が昔から好きなのでコードを書いていて楽しいです。

<!-- more -->

## PubNubの特徴

### Data Stream Network

世界中にデータセンターがありスケールするData Stream Networkを持っているのが特徴です。信頼性が高く、レイテンシの低いリアルタイムメッセージのためのCDNに近いようです。データセンターがダウンしても、トラフィックは他のデータセンターに迂回されるためユーザーは気づかないそうです。

### 豊富なSDK

[SDKが豊富](http://www.pubnub.com/developers/)で60種類以上あります。IoTデバイス向けには[Arduino](https://github.com/pubnub/arduino)や[Raspberry Pi](https://github.com/pubnub/c)などの他にも、[mbed](https://github.com/pubnub/mbed)や[Kinoma Create](https://github.com/pubnub/kinoma)のSDKも揃っています。

## Ember.js 101

PubNubのブログに投稿されている、[Ember.js 101: From Zero to Ember in PubNub Seconds](http://www.pubnub.com/blog/emberjs-101-from-zero-to-ember-in-pubnub-seconds/)のチュートリアルを進めていきます。

### Docker開発環境

Node.jsのDocker開発環境を使います。Bowerが必要なので[dockerfile/nodejs-bower-gulp](https://registry.hub.docker.com/u/dockerfile/nodejs-bower-gulp/)のイメージを使います。テスト用のため使い捨てのコンテナを起動します。

``` bash
$ docker pull dockerfile/nodejs-bower-gulp
$ docker run -it --rm dockerfile/nodejs-bower-gulp
```

Node.jsとnpmのバージョンを確認します。

``` bash
$ node -v
v0.10.35
$ npm -v
2.2.0
```

### Ember.js SDKのインストール

パッケージマネージャのBowerを使い[pubnub-ember](https://github.com/pubnub/pubnub-ember/)をインストールします。このDockerコンテナはrootユーザーでログインしているので、`--allow-root`フラグを付けます。

``` bash
$ bower install pubnub-ember --allow-root
bower not-cached    git://github.com/pubnub/pubnub-ember.git#*
bower resolve       git://github.com/pubnub/pubnub-ember.git#*
bower download      https://github.com/pubnub/pubnub-ember/archive/v0.1.0-beta.1.tar.gz
bower extract       pubnub-ember#* archive.tar.gz
bower resolved      git://github.com/pubnub/pubnub-ember.git#0.1.0-beta.1
bower install       pubnub-ember#0.1.0-beta.1

pubnub-ember#0.1.0-beta.1 bower_components/pubnub-ember
```

## チャットアプリの作成

### index.html

最初に今回作成するチャットアプリのindex.htmlです。オリジナルは[GitHubのリポジトリ](https://github.com/pubnub/pubnub-ember/blob/master/site/examples/chat/index.html)にあります。Ember.jsのバージョンやDEPRECATEDになったコードなど、若干修正しています。

``` index.html
<!doctype html>
<html>
  <head>
    <script src="http://code.jquery.com/jquery-1.11.2.min.js"></script>
    <script src="http://cdn.pubnub.com/pubnub.min.js"></script>
    <script src="http://builds.handlebarsjs.com.s3.amazonaws.com/handlebars-v2.0.0.js"></script>
    <script src="http://builds.emberjs.com/release/ember.js"></script>
    <script src="bower_components/pubnub-ember/lib/pubnub-ember.js"></script>
    <link rel="stylesheet" href="//netdna.bootstrapcdn.com/bootstrap/3.3.2/css/bootstrap.min.css">
  </head>
  <body>
    <script type="text/x-handlebars" data-template-name="application">
      <div class="container">
        <h4>Online Users</h4>
        <ul>
        &#123;&#123; #each user in users }}
          <li>&#123;&#123; user }}</li>
        &#123;&#123; /each }}
        </ul>
        <br />
        <h4>Chat History(&#123;&#123; messages.length }})</h4>
        <form>
          &#123;&#123;input type="text" value=new_message placeholder="Enter a message"}}          
          <input type="submit" &#123;&#123;action 'publish'}} />
        </form>
        <br />
        <div class="well">
          <ul>
          &#123;&#123; #each message in messages }}
            <li>&#123;&#123; message }}</li>
          &#123;&#123; /each }}
          </ul>
        </div>
      </div>
    </script>
    <script>
      window.Mini = Ember.Application.create();
      var user_id = "User " + Math.round(Math.random() * 1000);

      Mini.PubNub = PubNubEmber.extend({
        cfg: {
          subscribe_key: 'xxx',
          publish_key: 'xxx',
          uuid: user_id
        }
      });

      Mini.ApplicationController = Ember.Controller.extend({
        needs: ['pubnub:main'],
        channel: 'The EmberJS Channel',
        new_message: '',
        user_id: user_id,
        messages: Ember.ArrayProxy.create({content: Ember.A(['Welcome to The EmberJS Channel']) }),
        users: Ember.ArrayProxy.create({content: Ember.A([]) }),
        init: function() {
          var pn = this.get('pubnub');
          var chan = this.get('channel');
          var self = this;

          // Subscribe to the Channel
          pn.emSubscribe({ channel: chan });

          // Register for message events
          pn.on(pn.emMsgEv(chan), function(payload){
            self.get('messages').pushObject(payload.message);
          });

          // Register for presence events
          pn.on(pn.emPrsEv(chan), function(payload){
            self.get('users').set('content', pn.emListPresence(chan));
          });

          // Pre-Populate the user list (optional)
          pn.emHereNow({ channel: chan });

          // Populate messae history (optional)
          pn.emHistory({
            channel: chan,
            count: 500
          });
        },
        actions: {
          // set up an Ember Action to publish a message
          publish: function(){
            this.get('pubnub').emPublish({
              channel: this.get('channel'),
              message: "[" + this.get('user_id') + "] " + this.get('new_message')
            });
            this.set('new_message','');
          }
        }
      });
    </script>
  </body>
</html>
```

### headerのscriptタグ

```js:index.html
...
    <script src="bower_components/pubnub-ember/lib/pubnub-ember.js"></script>
```

`bower_components/pubnub-ember/pubnub-ember.js`は bower installでインストールしたディレクトリです。


### Handlebarsテンプレート

Ember.jsのテンプレートは[Handlebars](http://handlebarsjs.com/)が統合されています。今回のテンプレートはindex.htmlのscriptタグの中にインラインで記述しています。

```html index.html
  <body>
    <script type="text/x-handlebars" data-template-name="application">
      <div class="container">
        <h4>Online Users</h4>
        <ul>
        &#123;&#123;#each user in users}}
          <li>&#123;&#123;user}}</li>
        &#123;&#123;/each}}
...
      </div>
    </script>
  </body>
```

### PubNubEmberサービスの作成

PubNub Developer Portalにログインして、アカウントに紐付いたsubscribe_keyとpublish_keyを確認して記述します。

```html index.html
    <script>
      window.Mini = Ember.Application.create();
      var user_id = "User " + Math.round(Math.random() * 1000);
      Mini.PubNub = PubNubEmber.extend({
        cfg: {
          subscribe_key: 'xxx',
          publish_key: 'xxx',
          uuid: user_id
        }
      });
```

### ApplicationControllerの作成

Ember.js controllerオブジェクトにアプリケーションロジックを記述していきます。コントローラーにはビューで使用するコレクションなどの変数や関数を定義します。コレクションに使っているArrayProxyクラスは予め用意されています。プロパティに変化があると、ビューに自動的に反映してくれます。

### ApplicationControllerのinit関数

イベントリスナーの登録と、イベントハンドラ関数の定義の簡単な説明です。

* emSubscribe関数
 * アプリ用のsubscriptionチャンネルを作成する
 * channel名はEmber.Controller.extendの最初で定義している

* emMsgEv関数
 * messageイベントハンドラをイベントにバインドする
 * PubNub Ember.jsライブラリはチャンネルから受信したイベントを Ember.jsのイベントに変換してくれる
 * messageを受信したら、controllerのmessagesコレクションにpushする

* emPrsEv関数
 * presenseイベントに、イベントリスナを登録する
 * コントローラーのusersコレクションを動的にアップデートする

* emHereNow関数
 * presenceイベントをfireする
 * 登録したpresenceイベントハンドラよって処理される

* emHistory関数
 * messageイベントをfireする
 * 登録したmessaegsイベントハンドラによって処理される

## 確認

### local-web-serverの起動

[local-web-server](https://www.npmjs.com/package/local-web-server)はNode.js製の開発用HTTPサーバーです。local-web-serverを起動します。

``` bash
$ npm install -g local-web-server
```

ブラウザで確認します。

http://172.17.4.110:8000

![storage-is-not-enabled.png](/2015/01/26/pubnub-ember-tutorials-emberjs-101/storage-is-not-enabled.png)

画面にメッセージが表示されていますが、Storage & Playbackサービスが有効になっていないので、フォームをsubmitしてもメッセージは保存されません。


### Storage & Playbackサービスを有効にする

PubNub Developer Portalにログインして、FreaturesからStorage & Playbackのセクションに移動します。

![features-storage.png](/2015/01/26/pubnub-ember-tutorials-emberjs-101/features-storage.png)


addボタンを押して有効にします。

![enable-storage-feature.png](/2015/01/26/pubnub-ember-tutorials-emberjs-101/enable-storage-feature.png)

[料金表](http://www.pubnub.com/pricing/)によると、FREEプランの30日トライアルは1日分のメッセージを保存してくれるそうです。

画面からsubmitしたメッセージはブラウザをリフレッシュしても消えずに、Storageに保存されました。

![storage-is-enabled.png](/2015/01/26/pubnub-ember-tutorials-emberjs-101/storage-is-enabled.png)



