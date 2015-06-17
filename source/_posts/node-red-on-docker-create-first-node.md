title: "Node-RED on Docker - Part4: はじめてのnodeを作成する"
date: 2015-02-14 00:04:50
tags:
 - Docker
 - Node-RED
 - Nodejs
description: Node-REDのCreating your first nodeを読むと自分でnodeを定義できるので慣れてくるとNode-REDの可能性がいろいろ見えてきます。MasheryやWSO2のESBのような使い方ができます。以前開発で使っていたDataSpider Servistaを利用してセンサーデータハッカソンも開催されたようです。Node-REDはNode.jsで書けるのでアイデア次第でいろいろ使えそうです。
---

Node-REDの[Creating your first node](http://nodered.org/docs/creating-nodes/first-node.html)を読むと自分でnodeを定義できるので慣れてくるとNode-REDの可能性がいろいろ見えてきます。[Mashery](http://www.mashery.com/api-management)や[W2O2](http://wso2.com/landing/internet-of-things/)のESBのような使い方ができます。以前開発で使っていた[DataSpider Servista](http://dataspider.appresso.com/)を利用して[センサーデータハッカソン](http://kokucheese.com/event/index/242997/)も開催されたようです。Node-REDはNode.jsで書けるのでアイデア次第でいろいろ使えそうです。


<!-- more -->

## nodeの作り方

[前回](/2015/02/13/node-red-on-docker-build-error/)Dockerホストにカスタムのnodeを追加するディレクトリを`/opt/nodes`に作成し、settings.jsのnodesDirに指定しました。このディレクトリにjsとhtmlファイルを作成します。

* js:   nodeのロジックを記述
* html: nodeの設定を記述

## lower-caseのサンプル

[Creating your first node](http://nodered.org/docs/creating-nodes/first-node.html)にあるlower-caseのサンプルを実装してみます。

jsファイルにロジックを記述します。

``` js /opt/nodes/99-lower-case.js
module.exports = function(RED) {
  function LowerCaseNode(config) {
    RED.nodes.createNode(this,config);
    var node = this;
    this.on('input', function(msg) {
      msg.payload = msg.payload.toLowerCase();
      node.send(msg);
    });
  }
  RED.nodes.registerType("lower-case",LowerCaseNode);
}
```

htmlファイルにnodeの設定やアイコンの指定を記述します。

``` html /opt/nodes/99-lower-case.html
<script type="text/javascript">
  RED.nodes.registerType('lower-case',{
    category: 'function',
    color: '#a6bbcf',
    defaults: {
      name: {value:""}
    },
    inputs:1,
    outputs:1,
    icon: "file.png",
    label: function() {
      return this.name||"lower-case";
    }
  });
</script>

<script type="text/x-red" data-template-name="lower-case">
  <div class="form-row">
    <label for="node-input-name"><i class="icon-tag"></i> Name</label>
    <input type="text" id="node-input-name" placeholder="Name">
  </div>
</script>

<script type="text/x-red" data-help-name="lower-case">
  <p>A simple node that converts the message payloads into all lower-case characters</p>
</script>
```

## Node-REDの画面

jsとhtmlを作成してからDockerコンテナを起動します。

``` bash
$ docker run --rm \
  --name node-red \
  -p 1880:1880 \
  -v /opt/nodes:/data/nodes \
  node-red 
```

Node-REDの画面を開くとfunctionのカテゴリにlower-caseのnodeが追加されました。injectとdebugのnodeを前後につなげます。injectのpayloadには大文字でHELLOと入力しました。deploy後にinject nodeの左のボタンを押すとフローが実行されます。debugにはpayloadの大文字のHELLOが小文字のhelloになって出力されました。

![lower-case-node.png](/2015/02/14/node-red-on-docker-create-first-node/lower-case-node.png)

