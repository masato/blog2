title: "ES6で書くIsomorphicアプリ入門 - Part1: リソース"
date: 2015-07-02 12:44:02
tags:
 - ES6
 - Isomorphic
 - Koa
 - Express
 - Hapijs
 - React
 - Webpack
description: サーバーサイドはio.jsとKoa、クライアントサイドはReactを使って、ES6で書くIsomorphicアプリを開発してみようと思います。まずはBoilerplateやチュートリアル、ライブラリなどのリソースを集めてみました。Koa、Reactに加えてBabel、Webpack or Browserifyというのが基本的なアーキテクチャになりそうです。
---

サーバーサイドは[io.js](https://iojs.org/en/index.html)と[Koa](http://koajs.com/)、クライアントサイドは[React](http://facebook.github.io/react/)を使って、[ES6](https://iojs.org/ja/es6.html)で書くIsomorphicアプリを開発してみようと思います。まずはBoilerplateやチュートリアル、ライブラリなどのリソースを集めてみました。。Koa、Reactに加えて[Babel](https://babeljs.io/)、[Webpack](http://webpack.github.io/) or [Browserify](http://browserify.org/)というのが基本的なアーキテクチャになりそうです。


<!-- more -->

## The React.js Way

ハンガリーの[RisingStack](http://risingstack.com/)のブログに`The React.js Way`というポストがあります。簡単なReactのチュートリアルとFluxアーキテクチャの説明が書いてあります。リポジトリもそれぞれ公開されているので手を動かしながら勉強ができます。

* [The React.js Way: Getting Started Tutorial](http://blog.risingstack.com/the-react-way-getting-started-tutorial)
* [react-way-getting-started](https://github.com/RisingStack/react-way-getting-started)
* [The React.js Way: Flux Architecture with Immutable.js](http://blog.risingstack.com/the-react-js-way-flux-architecture-with-immutable-js/)
* [react-way-immutable-flux](https://github.com/RisingStack/react-way-immutable-flux)

### 構成

* [Express](http://expressjs.com/): サーバーサイド
* [React](http://facebook.github.io/react/): クライアントサイド
* [Babel](https://babeljs.io/): コンパイラ
* [Webpack](http://webpack.github.io/): モジュール管理
* [Flux](http://facebook.github.io/flux/): Flux
* [Immutable.js](https://facebook.github.io/immutable-js/): コレクション
* [Jest](https://facebook.github.io/jest/): テスト


## React Starter Kit

ロシアの[Kriasoft](http://www.kriasoft.com/)が開発しているReactの[テンプレート](https://github.com/kriasoft/react-starter-kit)です。RisingStackのチュートリアルと似ていますが、[BrowserSync](http://www.browsersync.io/)など[package.json](https://github.com/kriasoft/react-starter-kit/blob/master/package.json)を読むといまどきの開発環境を積極的に使っています。

* [React Starter Kit](http://www.reactstarterkit.com/)

### 構成

* [Express](http://expressjs.com/): サーバーサイド
* [React](http://facebook.github.io/react/): クライアントサイド
* [Babel](https://babeljs.io/): コンパイラ
* [Flux](http://facebook.github.io/flux/): Flux
* [Webpack](http://webpack.github.io/): モジュール管理
* [Jest](https://facebook.github.io/jest/): テスト


## React Isomorphic Starterkit

[React Isomorphic Starterkit](https://github.com/RickWong/react-isomorphic-starterkit)はサーバーサイドに[Hapi.js](http://hapijs.com/)を採用しています。RisingStackの[Hapi on Steroids - Using Generator Functions with Hapi](http://blog.risingstack.com/hapi-on-steroids-using-generator-functions-with-hapi/)のポストにもHapi.jsをES6で書くサンプルがあります。


* [React Isomorphic Starterkit](https://github.com/RickWong/react-isomorphic-starterkit)
* [How to use ES6 generators with Hapi.js <3](https://gist.github.com/grabbou/ead3e217a5e445929f14)

### 構成

* [Hapi.js](http://hapijs.com/): サーバーサイド
* [React](http://facebook.github.io/react/): クライアントサイド
* [Babel](https://babeljs.io/): コンパイラ
* [Webpack](http://webpack.github.io/): モジュール管理


## Este.js

[Este.js](https://github.com/este/este)はAngularJSが苦手な方に良いみたいです。フルスタックの`universal funtional` webアプリのためのスタータキットです。煽りが刺激的です。[Learn React.js](http://learn-reactjs.com/)によるとチェコの方が開発されています。

* [Este.js](https://github.com/este/este)
* [Learn React.js](http://learn-reactjs.com/)
* [Este.js Framework](https://medium.com/este-js-framework)
* [este-todomvc](https://github.com/steida/este-todomvc)

* [What’s wrong with Angular 1](https://medium.com/este-js-framework/whats-wrong-with-angular-js-97b0a787f903)
* [What I would recommend instead of Angular.js?](https://medium.com/este-js-framework/what-i-would-recommend-instead-of-angular-js-62b057d8a9e)

 
### 構成

* [Express](http://expressjs.com/): サーバーサイド
* [React](http://facebook.github.io/react/): クライアントサイド
* [Babel](https://babeljs.io/): コンパイラ
* [Webpack](http://webpack.github.io/): モジュール管理
* [Flux](http://facebook.github.io/flux/): Flux
* [Immutable.js](https://facebook.github.io/immutable-js/): コレクション


## react-isomorphic-boilerplate

Joseph Furlott氏がポストしている2つのチュートリアルは非常にわかりやすく、[react-isomorphic-boilerplate](https://github.com/jmfurlott/react-isomorphic-boilerplate)のウォークスルーになっています。特に[Webpack](http://webpack.github.io/)と[react-router](https://github.com/rackt/react-router)の使い方がとても勉強になります。

* [react-isomorphic-boilerplate](https://github.com/jmfurlott/react-isomorphic-boilerplate)

* [Tutorial: Setting Up a Simple Isomorphic React app](http://jmfurlott.com/tutorial-setting-up-a-simple-isomorphic-react-app/)
* [Tutorial: Setting Up a Single Page React Web App with React-router and Webpack]( http://jmfurlott.com/tutorial-setting-up-a-single-page-react-web-app-with-react-router-and-webpack/)

### 構成

* [Express](http://expressjs.com/): サーバーサイド
* [React](http://facebook.github.io/react/): クライアントサイド
* [Babel](https://babeljs.io/): コンパイラ
* [Webpack](http://webpack.github.io/): モジュール管理
* [react-router](https://github.com/rackt/react-router): Router
* [Jade](http://jade-lang.com/): テンプレート


## UniversalJS Boilerplate

[UniversalJS Boilerplate](https://github.com/carlosazaustre/universal-js-boilerplate)は`universal (isomorphic)`なwebアプリのためのboilerplateです。

* [UniversalJS Boilerplate](https://github.com/carlosazaustre/universal-js-boilerplate)


### 構成

* [Express](http://expressjs.com/): サーバーサイド
* [React](http://facebook.github.io/react/): クライアントサイド
* [Babel](https://babeljs.io/): コンパイラ
* [Browserify](http://browserify.org/): モジュール管理



## isomorphic-flux-boilerplate

[ES6 Isomorphic Flux/ReactJS Boilerplate](https://github.com/iam4x/isomorphic-flux-boilerplate)はKoaとReactを使うIsomorphicアプリのBoilerplateです。Fluxの実装には[Alt](http://alt.js.org/)を使っています。


* [ES6 Isomorphic Flux/ReactJS(Boilerplate](https://github.com/iam4x/isomorphic-flux-boilerplate)
* [demo](http://isomorphic.iam4x.fr/)

### 構成

* [Koa](http://koajs.com/): サーバーサイド
* [React](http://facebook.github.io/react/): クライアントサイド
* [Babel](https://babeljs.io/): コンパイラ
* [Webpack](http://webpack.github.io/): モジュール管理
* [Alt](http://alt.js.org/): Flux
* [Iso](https://github.com/goatslacker/iso): Isomorphicのヘルパー


## ES6 boilerplate SPA

[ES6 boilerplate SPA](http://isomorphic.iam4x.fr/)は、ES6でSPAを書くためのBoilerplateです。簡単なhello-worldアプリのサンプルになっています。


* [es6-boilerplate](https://github.com/greim/es6-boilerplate)
* [Boilerplate single page app using ES6, Koa and React.](http://www.reddit.com/r/javascript/comments/2zun7t/boilerplate_single_page_app_using_es6_koa_and/)


### 構成

* [Koa](http://koajs.com/): サーバーサイド
* [React](http://facebook.github.io/react/): クライアントサイド
* [Babel](https://babeljs.io/): コンパイラ
* [Browserify](http://browserify.org/): モジュール管理


## koa-react-full-example

[koa-react-full-example](https://github.com/dozoisch/koa-react-full-example)は、KoaやReactに加えて、[Passport](https://www.npmjs.com/package/passport)の認証、[Mongoose](http://mongoosejs.com/)のMongoDBなども使う、本格的なアプリのサンプルです。

### 構成

* [Koa](http://koajs.com/): サーバーサイド
* [React](http://facebook.github.io/react/): クライアントサイド
* [Babel](https://babeljs.io/): コンパイラ
* [Webpack](http://webpack.github.io/): モジュール管理


## Deku

[Deku](https://github.com/dekujs/deku)はReactのクライアントサイドのalternativeです。サーバーサイドとの依存関係はありませんが、READMEにはKoaを使う例が書いてあります。[TodoMVC](https://github.com/dekujs/todomvc)がサンプル実装です。


* [Deku](https://github.com/dekujs/deku)
* [TodoMVC](https://github.com/dekujs/todomvc)

### 構成

* [Deku](https://github.com/dekujs/deku): クライアントサイド
* [Babel](https://babeljs.io/): コンパイラ
* [Duo](http://duojs.org/): パッケージ管理
* [Browserify](http://browserify.org/): モジュール管理

## horse

[horse](https://github.com/reddit/horse)は[reddit](reddit)が開発している、Isomorphicアプリを開発するためのヘルパーパッケージです。[horse-react](https://github.com/reddit/horse-react)がサンプル実装です。


* [horse](https://github.com/reddit/horse)
* [horse-react](https://github.com/reddit/horse-react)

### 構成

* [Koa](http://koajs.com/): サーバーサイド
* [React](http://facebook.github.io/react/): クライアントサイド
* [Babel](https://babeljs.io/): コンパイラ
