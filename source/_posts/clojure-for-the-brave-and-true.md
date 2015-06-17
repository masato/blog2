title: "Clojure for the Brave and Trueで勉強しながらデータ分析環境を考える"
date: 2014-09-28 21:25:51
tags:
 - AnalyticSandbox
 - Clojure
 - IPythonNotebook
 - GorillaREPL
description: Clojure for the Brave and Trueがオンラインで公開されています。Leanpubから電子書籍として購入することもできます。Clojure以外の他の書籍と比べてもかなり読みやすく初学者向きです。疑問に感じるところにはすぐフォローが丁寧に書かれているので、躓きやすいところがわかり学習が捗ります。in Actionで手を動かしながら勉強するのが好きなので、Emacsの使い方から書かれているのもうれしいです。
---

[Clojure for the Brave and True](http://www.braveclojure.com/)がオンラインで公開されています。[Leanpub](https://leanpub.com/clojure-for-the-brave-and-true)から電子書籍として購入することもできます。

Clojure以外の他の書籍と比べてもかなり読みやすく初学者向きです。疑問に感じるところにはすぐフォローが丁寧に書かれているので、躓きやすいところがわかり学習が捗ります。`in Action`で手を動かしながら勉強するのが好きなので、Emacsの使い方から書かれているのもうれしいです。

<!-- more -->

### Clojureのデータ分析関連書籍

少しずつClojureのデータ構造を理解していくと、データ分析ツールとして人気が高いのもわかる気がします。
PacktからClojureのデータ分析関連書籍が3冊出版されていました。

* [Clojure Data Analysis Cookbook](https://www.packtpub.com/big-data-and-business-intelligence/clojure-data-analysis-cookbook)
* [Mastering Clojure Data Analysis](https://www.packtpub.com/big-data-and-business-intelligence/mastering-clojure-data-analysis)
* [Clojure for Machine Learning](https://www.packtpub.com/big-data-and-business-intelligence/clojure-machine-learning)

最近Clojure本ばかり買っている気がしますが、久しぶりにプログラミングが楽しいです。

### Dockerデータ分析環境

REPL環境は[CIDER](/2014/09/24/ac-nrepl-deprecated-using-ac-cider/)と[ClojureScript REPL](/2014/09/25/clojurescript-repl-examples/)をDocker上に構築しています。

もう少し[IPythonNotebook](/2014/07/11/docker-analytic-sandbox-anaconda-ipython-notebook/)みたいにブラウザから実行できるREPLを探していると、[Gorilla REPL](https://github.com/JonyEpsilon/gorilla-repl)を見つけたので、Dockerデータ分析環境を作ってみようと思います。




