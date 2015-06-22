title: "Reagent入門 - Part4: SPAとフォームの要件"
date: 2015-06-22 20:19:09
tags:
 - SPA
 - Reagent
 - Clojure
 - ClojureScript
 - Rente
 - Sente
 - Figwheel
description: Reagentの勉強が1ヶ月ほど中断してしまいました。ClojureScriptの世界では依存パッケージのアップデートやReagentの使い方も新しくなっていると思います。Reactのようなライブラリを使う場合は特に動きが速いので、すぐに取り残されてしまいます。
---

[Reagent](https://github.com/reagent-project/reagent)の勉強が1ヶ月ほど中断してしまいました。ClojureScriptの世界では依存パッケージのアップデートやReagentの使い方も新しくなっていると思います。Reactのようなライブラリを使う場合は特に動きが速いので、すぐに取り残されてしまいます。

<!-- more -->

## SPAの要件

SPAのフォームに必要な要件を考えてみました。

* [Bootstrap](http://getbootstrap.com/)に対応
* AjaxやWebSocketを使ったデータバインディング
* APIや使い方のドキュメントが豊富にあること
* よく使うコンポーネントがデフォルトであること
* [React](http://facebook.github.io/react/)にこだわりませんが、リアクティブであること

REST APIサーバーは[Hapi.js](http://hapijs.com/)や[actionhero.js](http://www.actionherojs.com/)で実装したアプリを使おうと思います。まだClojureのフルスタックの構成が馴染めていないので、クライアントとサーバーを疎結合にして実装を切り換えることができるのもSPAの良いところだと思います。

## Reagentで使えるフォームライブラリ

[Reagent](https://github.com/reagent-project/reagent)に対応したフォームのライブラリにはコンポーネント単位のものからReagentを拡張したフルスタックフレームワークまで結構豊富にあります。

### reagent-forms

[reagent-forms](https://github.com/reagent-project/reagent-forms)はオフィシャルのReagentプロジェクトのライブラリです。[Bootstrap](http://getbootstrap.com/)に対応しています。

### re-com

[re-com](https://github.com/Day8/re-com)は[re-frame](https://github.com/Day8/re-frame)の姉妹プロジェクトです。それぞれ独立して使えますがあわせて使うと良いみたいです。Bootstrapに対応しています。[Material Design Iconic Font](http://zavoloklom.github.io/material-design-iconic-font/icons.html)も付いてます。[ドキュメント](http://re-demo.s3-website-ap-southeast-2.amazonaws.com/)がしっかりと書かれているので信頼できます。

### Rente

[Rente](https://github.com/enterlab/rente)は[core.async](https://github.com/clojure/core.async)とWebSocket/Ajaxライブラリの[Sente](https://github.com/ptaoussanis/sente)にReagentに対応させたフレームワークです。Bootstrap対応です。


core.asyncとWebSocket/Ajaxライブラリの[Sente](https://github.com/ptaoussanis/sente)をReagentに対応させたフレームワークのようです。Bootstrap対応です。

[前回](/2015/06/19/reagent-client-server-connection/)調べたサーバーとの通信方法を考えると、Renteが一番よさそうな感じです。
