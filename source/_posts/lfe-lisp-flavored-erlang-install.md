title: "Lisp Flavored Erlang (LFE) のインストール"
date: 2015-05-29 22:36:55
tags:
 - LFE
 - Elixir
 - Erlang
 - Lisp
description: ElixirのおかげでErlangに注目が集まっています。今までErlangを使ってみたくても、特殊に見える文法に戸惑っていたプログラマに、Ruby風なシンタックスのおかげで敷居が低くなったと思います。一方のLisp Flavored Erlang (LFE)は名前の通りLisp風に書けるErlangです。誰がうれしいのか一瞬よくわかりませんが、Lispには何か避けられない魅力があります。かつてもっと文明的だった時代のエレガントな武器です。
---

[Elixir](http://elixir-lang.org/)のおかげで[Erlang](http://www.erlang.org/)に注目が集まっています。今までErlangを使ってみたくても、特殊に見える文法に戸惑っていたプログラマに、Ruby風なシンタックスのおかげで敷居が低くなったと思います。一方の[Lisp Flavored Erlang (LFE)](http://lfe.io/)は名前の通りLisp風に書けるErlangです。誰がうれしいのか一瞬よくわかりませんが、Lispには何か避けられない魅力があります。[かつてもっと文明的だった時代のエレガントな武器です](https://xkcd.com/297/)。

<!-- more -->

ScalaやClojureとったJVM上で動作する新しい言語も関数型言語の特徴を持つことが多く、不変性や並行性、分散処理を重要視しています。時代とともに純粋なプログラミング言語としてJavaやErlangの魅力は薄れつつありますが、クリティカルなエンタープライズの分野で長年の信頼と実績を持つJVMやErlang VMの性能を享受できるのは魅力的です。

## インストール

GitHubのリポジトリにあるUbutntu版の[Dockerfile](https://github.com/lfex/dockerfiles/blob/master/ubuntu/)を見てもソースからErlangをインストールするのはとても面倒です。セットアップが面倒な環境ではDockerが便利です。

### Debian版

Dockerイメージの[lfex/lfe](https://registry.hub.docker.com/u/lfex/lfe/)はDebianがベースになっています。`lfe`コマンドを実行するとREPLが起動します。Erlang/OTP 17、LFE V6.2とちょっとバージョンが古いようです。

```bash
$ docker pull lfex/ubuntu
$ docker run -i -t lfex/ubuntu lfe
Erlang/OTP 17 [erts-6.2] [source] [64-bit] [async-threads:10] [kernel-poll:false]

LFE Shell V6.2 (abort with ^G)
```

### Ubuntu版

DockerイメージはDebianの他にも[リポジトリ](https://github.com/lfex/dockerfiles)にはいろいろなOSで作成されたものがあります。[Raspbian](https://www.raspbian.org/)のあります。UbuntuはErlang/OTP 18、LFE v0.9.1と新しいバージョンなのでこちらで開発環境を作ろうと思います。


```bash
$ docker pull lfex/ubuntu
$ docker run -i -t lfex/ubuntu lfe
Erlang/OTP 18 [erts-7.0] [source-4d83b58] [64-bit] [async-threads:10] [hipe] [kernel-poll:false]

         (
     (    )  )
      )_.(._(
   .-(   \\  )-.       |   A Lisp-2+ on the Erlang VM
  (     / \\    )      |   Docs: http://docs.lfe.io/
  |`-.._____..-';.     |
  |         g  (_ \    |   Type (help) for usage info.
  |        n    || |   |
  |       a     '/ ;   |   Source code:
  (      l      / /    |   http://github.com/rvirding/lfe
   \    r      ( /     |
    \  E      ,y'      |   LFE v0.9.1
     `-.___.-'         |   LFE Shell V7.0 (abort with G^)

```

## 開発環境

適当なディレクトリを作成します。リポジトリは[こちら](https://github.com/masato/docker-lfe)です。

```bash
$ cd ~/lfe_apps
$ tree
.
├── Dockerfile
├── README.md
├── docker-compose.yml
└── sample-module.lfe
```

Dockerfileは[lfex/ubuntu](https://registry.hub.docker.com/u/lfex/ubuntu/)をベースイメージにします。WORKDIRを作成するだけです。

```bash ~/lfe_apps/Dockerfile
FROM lfex/ubuntu
MAINTAINER Masato Shimizu <ma6ato@gmail.com>

WORKDIR /app
COPY . /app
```

docker-compose.ymlではローカルにビルドしたイメージを使い、`lfe`と`lfescript`コマンドを使うサービスを定義します。

```yaml ~/lfe_apps/docker-compose.yml
lfe: &defaults
  image:  masato/lfe
  volumes:
    - .:/app
  entrypoint: lfe
lfescript:
  <<: *defaults
  entrypoint: lfescript
```

ローカルにDockerイメージをビルドしておきます。

```bash
$ docker build -t masato/lfe .
```

## 簡単な使い方

### REPL

さきほど実行したように`lfe`コマンドを実行するとREPLが起動します。[2 The LFE REPL](http://lfe.gitbooks.io/quick-start/content/2.html)からいくつか写経してみます。

```bash
$ docker-compose run --rm lfes
>
```

Hello Worldと簡単なかけ算です。

```el
> (io:format "Hello, World!~n")
Hello, World!
ok
> (* 2 21)
42
```

my-list変数に連番をバインドします。listsはErlangの組み込み関数のモジュールです。seq関数で連番を作成します。

```el
> (set my-list (lists:seq 1 6))
(1 2 3 4 5 6)
```

同じくlistsモジュールのsum関数で変数にバインドした連番の合計した結果の21に、2を乗算します。

```el
> (* 2 (lists:sum my-list))
42
```

もっと関数型言語っぽく`lists:foldl`関数を使い、my-list変数に累計する無名関数を適用して畳み込み(reduce)した結果の21に、2を乗算します。

```el
> (* 2 (lists:foldl (lambda (n acc) (+ n acc)) 0 my-list))
42
```

Ctrl-Cを2回押すとREPLが終了します。

### lfescript

[lfescript](https://github.com/rvirding/lfe/blob/develop/doc/lfescript.txt)はスクリプトとして書いてコンパイルせずに実行することができます。リポジトリにある[sample-lfescript](https://github.com/rvirding/lfe/blob/develop/examples/sample-lfescript)のサンプルを実行してみます。関数型言語のサンプルによくあるfactorial(階乗)の計算です。スクリプトの実行にはスクリプトの先頭にshebangでlfescriptが必要です。

```el sample-module.lfe
#! /usr/bin/env lfescript

(defun main
  ([(list string)]
   (try
     (let* ((n (list_to_integer string))
                    (f (fac n)))
           (: lfe_io format '"factorial ~w = ~w\n" (list n f)))
     (catch
       ((tuple _ _ _) (usage)))))
  ([_] (usage)))

(defun fac
  ([0] 1)
  ([n] (* n (fac (- n 1)))))

(defun usage ()
  (let ((script-name (: escript script_name)))
    (: lfe_io format '"usage: ~s <integer>\n" (list script-name))))
}}}
```

6の階乗を計算してみます。`lfescript`に続いて実行するスクリプトファイル、引数を入力します。

```bash
$ docker-compose run --rm lfescript sample-module.lfe 6
factorial 6 = 720
```
