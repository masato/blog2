title: 'Introduction to Dart から見るDartの魅力'
date: 2014-06-17 00:29:45
tags:
 - Dart
 - Generics
description: 仮想マシンやインフラばかりいじっていて疲れてきたので、本業のプログラミングの勉強をします。AppleがSwiftを発表してから、また開発言語論が活発になってきたようです。Swiftは様子見なのですが、DartでChromeアプリやAndroidアプリを書いてみたいです。
---

仮想マシンやインフラばかりいじっていて疲れてきたので、本業のプログラミングの勉強をします。
AppleがSwiftを発表してから、また開発言語論が活発になってきたようです。
Swiftは様子見なのですが、DartでChromeアプリやAndroidアプリを書いてみたいです。

いつものようにDartの情報をググっていたら、[Introduction to Dart](http://www.slideshare.net/RameshNair6/introduction-to-dart-35252146)というスライドを見つけました。
Dartを選ぶ理由を挙げる資料のなかでも、とてもコンパクトにまとまっていて、Dartの魅力にあふれています。
[How I Learned to Stop Worrying, and Love Dart](http://mattbriggs.net/blog/2014/03/10/how-i-learned-to-stop-worrying/)もわかりやすいポストです。

関数型とオブジェクト指向のよいミックスでも、Scalaの複雑性がありません。
DOMの操作では、GWTで失敗したJavaで書くことの無理がなく、ここはJavaScriptのよいところが出ています。
大規模な分散システムを書く場合は、ScalaやElrangもよいですが、WebアプリはサーバーサイドもDartかなと思います。

このスライドで、一番気に入っているのは、Genericsです。最近はRubyばかり書いているので、なんかほっとしました。
静的型付けでコードの可読性や安全性を高めることができるのですが、実行時はオーバーヘッドなく高速なので、安心できるのだと思います。

型チェックは、プロダクションモードの`Dart VM`ではパフォーマンス重視のために無視されます。
型システムは、あくまで開発時にツールが静的解析するサポートや、プログラマーの意図を確認するためにあります。

``` dart
class Cache<T> {
  Map<String,T> _m;
  Cache() {
    _m = new Map<String, T>();
  }
  T get(String key)          => _m[key]; 
    set(String key, T value) => _m[key] = value;
}

void main() {
  varc = new Cache<String>();
  c.set('test', 'value');
  print(c.get('test') == 'value');
  c.set('test', 43); // fails in checked mode
}
```