title: "Promiseでwhileループを書いてみる"
date: 2015-09-02 15:24:32
tags:
 - ES6
 - Bluebird
 - Promise
 - iojs
 - Q
description: io.jsのES6では標準でPromiseが使えるようになりました。Async.jsで書いてきたコードも、これからはPromiseやGenerator、coなどのコルーチンに移行していきたいです。whileループの書き方がよくわからなかったので、Stack Overflowなどで調べてみました。
---

[io.js](https://iojs.org/ja/)のES6では標準でPromiseが使えるようになりました。[Async.js](https://github.com/caolan/async)で書いてきたコードも、これからはPromiseやGenerator、coなどのコルーチンに移行していきたいです。whileループの書き方がよくわからなかったので、Stack Overflowなどで調べてみました。

<!-- more -->


## リソース

以下のサイトを参考にしました。

* ["Infinite" promise chains, a bad thing? #477](https://github.com/petkaantonov/bluebird/issues/477)
* [Bluebird](https://github.com/petkaantonov/bluebird)
* [While loop with promises](http://stackoverflow.com/questions/17217736/while-loop-with-promises)
* [While loop using bluebird promises](http://stackoverflow.com/questions/29375100/while-loop-using-bluebird-promises)
* [Correct way to write loops for promise.](http://stackoverflow.com/questions/24660096/correct-way-to-write-loops-for-promise)


## Bluebirdの無限ループ

[Bluebird](https://github.com/petkaantonov/bluebird)は高機能なPromiseのライブラリです。ES6 Native Promiseにはないメソッドや、[コルーチン](https://github.com/petkaantonov/bluebird/blob/master/API.md#promisecoroutinegeneratorfunction-generatorfunction---function)も用意されています。["Infinite" promise chains, a bad thing? #477](https://github.com/petkaantonov/bluebird/issues/477)にwhileループの例がありました。

```js infinite-loop-bluebird.es6
var Promise = require('bluebird');
Promise.resolve(0).then(function loop(i) {
    return new Promise(function(resolve, reject) {
        console.log(i);
        resolve(i+1);
    })
    .delay(1000)
    .then(loop);
});
```

プログラムはio.jsの3.2.0、Bluebirdは2.9.34実行しました。

```bash
$ node -v
v3.2.0
```

実行結果

1秒間隔で数字がインクリメントされます。無限ループなので適当なところで止めます。

```bash
$ node infinite-loop-bluebird.es6
0
1
2
3
...
```


## ES6 Native Promiseの無限ループ

上記をES6 Native Promiseに書き換えてみます。ES6ネイティブにはBluebirdの[delay](https://github.com/petkaantonov/bluebird/blob/master/API.md#promisedelaydynamic-value-int-ms---promise)や、Qの[delay](https://github.com/kriskowal/q/wiki/API-Reference#qdelayms)がないので`setTimeout`で代用します。

```js infinite-loop-native.es6
Promise.resolve(0).then(function loop(i) {
    return new Promise(function(resolve, reject) {
        setTimeout(function() {
            console.log(i);
            resolve(i+1);
        }, 1000);
    })
    .then(loop);
});
```


実行結果

Bluebirdの結果と同じです。

```bash
$ node infinite-loop-native.es6
0
1
2
3
...
```


## ES6 Native Promiseのwhile ループ


[While loop with promises](http://stackoverflow.com/questions/17217736/while-loop-with-promises)にあるのは、[Q](https://github.com/kriskowal/q)の例ですがES6 Native Promiseに書き換えてみました。


```js while-loop-native
function action(i) {
    console.log(i);
    return {
        done: i > 9,
        value: i+1
    };
}

function loop(promise, fn) {
    return promise.then(fn).then(function(wrapper) {
        return !wrapper.done ? loop(Promise.resolve(wrapper.value), fn): wrapper.value;
    });
}

loop(Promise.resolve(0), action).then(function(value) {console.log('end: ' + value)});
```

実行結果

数字がインクリメントされ、`i > 9`がtrueになるとループが停止します。

```bash
$ node while-loop-native.es6

0
1
2
3
4
5
6
7
8
9
10
$ node while-loop-native.es6

0
1
2
3
4
5
6
7
8
9
10
end: 11
```

## Bluebirdのwhile ループ

[Memory leak trying to create a while loop with promises #502](https://github.com/petkaantonov/bluebird/issues/502)に再帰を使ったシンプルなループの例がありました。

```js just-recursion.es6
var Promise = require('bluebird');
(function loop(i) {
  if (i < 10) {
    return Promise.delay(100).then(function() {
      console.log(i);
      return i+1;
    }).then(loop);
  }
  return Promise.resolve(i);
})(0);
```

実行結果

```bash
$ node just-recursion.es6

0
1
2
3
4
5
6
7
8
9
10
```

もうひとつはBluebirdの[Promise.coroutine](https://github.com/petkaantonov/bluebird/blob/master/API.md#promisecoroutinegeneratorfunction-generatorfunction---function)を使う例です。ジェネレーターの中でwhileループを書きます。

```js coroutine-loop.es6
var Promise = require('bluebird');

function condition(i) {
    return i > 10;
}

function action(i) {
    console.log(i);
    return Promise.resolve(i+1);
}

var loop = Promise.coroutine(function* (condition, action, value) {
    while(!condition(value)) {
        value = yield action(value);
        yield Promise.delay(100);
    }
});

loop(condition, action, 0);
```

実行結果

```
$ node coroutine-loop.es6

0
1
2
3
4
5
6
7
8
9
10
```