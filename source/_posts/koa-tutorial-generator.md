title: "Koa入門 - Part3: ジェネレータなどKoaの特徴"
date: 2015-06-27 10:32:58
tags:
 - Nodejs
 - ES6
 - Koa
 - iojs
description: 前回までにKoaのアプリを開発するためのio.jsのDockerイメージを作成したり、ES6で書いたJavaScriptのコンパイルの方法を試しました。今回はジェネレータなどKoaで開発する場合に重要な特徴を見ていきます。
---

前回までにKoaのアプリを開発するための[io.jsのDocker](/2015/06/25/koa-tutorial-iojs-on-docker/)イメージを作成したり、[Babel](https://babeljs.io/)
でES6で書いたJavaScriptの[コンパイル](/2015/06/26/koa-tutorial-es6-babel-register/)を試しました。今回はジェネレータなどKoaで開発する場合に重要な特徴を見ていきます。

<!-- more -->

## リソース

[Koa](https://github.com/koajs/koa)のREADMEにGetting startedとして以下のサイトが紹介されていました。

* [workshop](https://github.com/koajs/workshop)
* [kick-off-koa](https://github.com/koajs/kick-off-koa)
* [examples](https://github.com/koajs/examples)

やはり最初のジェネレーター関数を理解するところです。ハンガリーの[RisingStack](http://risingstack.com/)というNode.jsの開発会社のブログを写経していきます。とてもわかりやすい英語で大変勉強になりました。強みにしているのがJavaScript/DevOps/IoTというのも親近感があります。

* [Getting Started with Koa, part 1 - Generators](http://blog.risingstack.com/introduction-to-koa-generators/)


## ジェネレータ関数

ジェネレータ関数はfunctionキーワードの後に`*`のアスタリスクを付けて定義します。

```js
function *foo() {}
```

この関数をコールするとイテレータオブジェクトが返ります。通常の関数と異なりコールしても実行されません。返されたイテレータに対して処理を実行します。

```js
> function *foo(arg) {}
> var bar = foo(123);
```

イテレータオブジェクト`bar`の`next()`メソッドをコールしてイテレートを開始します。`next()`をコールすると関数は開始するか、中断していた場合は次のポイントまで実行されます。この処理ではジェネレータの状態オブジェクトを返します。`value`プロパティは現在のイテレーションの値で、この場所でジェネレータは中断しています。もう一つはブール値の`done`プロパティです。こちらはジェネレータが終了したかを示します。

```js
> function *foo(arg) { return arg }
> var bar = foo(123);
> bar.next()
{ value: 123, done: true }
```

この例では処理の中断はしていないので`done`が`true`のオブジェクトがすぐに返ります。もしジェネレータの中で`return`があれば、最後のイテレータオブジェクト(`done`がtrue)が返ります。
次にジェネレータを中断してみます。関数をイテレーションする毎に`yield`が処理を中断したときの値を返します。つまり`yield`キーワードのところで処理は中断しています。

## yield

`next()`をコールするとジェネレータは開始して`yield`のところまで進みます。その場所で`value`と`done`を含むオブジェクトを返します。`value`は式(expression)の値です、数値や文字列など結果として値を返すものならなんでも良いです。

```js
function *foo() {
  var index = 0;
  while (index < 2) {
    yield index++;
  }
}

var bar = foo();
bar.next()
//{ value: 0, done: false }
bar.next()
//{ value: 1, done: false }
bar.next()
//{ value: undefined, done: true } 
```

`next()`を再び実行すると`yield`の結果が返り処理が再開します。イテレータオブジェクトからジェネレータの中で値を受け取ることもできます。`next(val))`のようにすると、処理が再開したときにジェネレータに値が返ります。

```js
> function *foo() {
...   var val = yield 'A';
...   console.log(val);
... }
> var bar = foo();
> console.log(bar.next());
{ value: 'A', done: false }
> console.log(bar.next('B'))
B
{ value: undefined, done: true }
```

## エラー処理

間違いを見つけた時はイテレータオブジェクトの`throw()`メソッドをコールするとジェネレータの中でエラーをcatchしてくれます。こうするとジェネレータの中のエラーハンドリングがとても良く書けます。

```js
function *foo() {
  try {
    x = yield 'asd B';
  } catch (err) {
    throw err;
  }
}

var bar = foo();
if (bar.next().value == 'B') {
  bar.throw(new Error("its' B!"));
}
```

## for...of

ES6のループにはジェネレータをイテレートするきにつかう`foo...of`ループがあります。イテレーションは`done`が`false`になるまで続きます。注意が必要なのはこのループを使う時には`next()`のコールに値を渡すことができません。ループは渡された値を無視します。

```js
function *foo() {
  yield 1;
  yield 2;
  yield 3;
}

for (v of foo()) {
  console.log(v);
}
```

## yield*

上記のようにyieldでは何でも扱えるため、ジェネレータも可能ですがその場合は`yield *`を使う必要があります。この処理をデリゲーションといいいます。別のジェネレータをデリゲートすることができるので、複数のネストされたジェネレータのイテレーションを、1つのイテレータオブジェクトを通して操作することができます。

```js
function *bar() {
  yield 'b';
}

function *foo () {
  yield 'a';
  yield *bar();
  yield 'c';
}

for (v of foo()) {
  console.log(v);
}
//a
//b
//c
```

## Thunks

`Thunks`はまた別のコンセプトですが、Koaを深く知るためには理解が必要です。主に別の関数のコールしやすくするために使われます。サンクは遅延評価とも呼ばれます。重要なことはNode.jsのコールバックを引数から関数のコールとして外に出せるということです。

```js
var read = function(file) {
  return function(cb) {
    require('fs').readFile(file, cb);
  }
}

read('package.json')(function (err, str) {});
}
```

[thunkify](https://github.com/tj/node-thunkify)という小さなモジュールは、普通のNode.jsの関数をthunkに変換してくれます。どう使うか疑問に思うかもしれませんが、コールバックをジェネレータの中に放り込むときに便利です。

```js
var thunkify = require('thunkify');
var fs = require('fs');
var read = thunkify(fs.readFile);

function *bar () {
  try {
    var x = yield read('input.txt');
  } catch (err) {
    throw err;
  }
  console.log(x);
}

var gen = bar();
gen.next().value(function (err, data) {
  if(err) gen.throw(err);
  gen.next(data.toString());
});
```

十分な時間をとってこの例をパーツ毎に理解することが必要なのは、Koaを使うときにとても重要なことがあるからです。ジェネレータの使い方が特にクールです。同期的なコードのシンプルさがありながら、エラー処理もしっかりしているのに、これは非同期で動いています。