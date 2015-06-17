title: "Elixir on Dockerで関数型言語をはじめよう - Part1: iExと文字列"
date: 2015-06-07 14:21:11
tags:
 - Erlang
 - Elixir
 - 関数型言語
 - DockerCompose
description: 先月ManningからElixir in Actionが出版されました。MEAPから積ん読だったのでそろそろ読み始めようと思います。最近はPhoenixというWebフレームワークも人気が出ています。Erlangがすっと入ってこなかったところと、Rubyで嫌いなところが、Elixirならメインの開発言語として使えるのか楽しみです。Elixirはドキュメントサイトが充実していて読みやすいです。まずはGetting StartedからIntroductionが動く環境を用意します。
---

先月Manningから[Elixir in Action](http://manning.com/juric/)が出版されました。MEAPから積ん読だったのでそろそろ読み始めようと思います。最近は[Phoenix](http://www.phoenixframework.org/)というWebフレームワークも人気が出ています。Erlangがすっと入ってこなかったところと、Rubyで嫌いなところが、Elixirで書くとメインの開発言語として使えるのか楽しみです。Elixirはドキュメントサイトが充実していて読みやすいです。まずはGetting Startedから[Introduction](http://elixir-lang.org/getting-started/introduction.html)が動く環境を用意します。

<!-- more -->

## プロジェクト

適当なところにプロジェクトを作成してdocker-compose.ymlを用意します。

```bash
$ cd ~/elixir_apps
$ tree
.
└── docker-compose.yml
```

Dockerイメージは[trenpixster/elixir](https://registry.hub.docker.com/u/trenpixster/elixir/)を使います。リポジトリは明記されていませんがたぶんこちら、[trenpixster/elixir-dockerfile](https://github.com/trenpixster/elixir-dockerfile)です。

```yml　~/elixir_apps/docker-compose.yml
elixir: &defaults
  image: trenpixster/elixir
  working_dir: /app
  volumes:
    - .:/app
  entrypoint: ["elixir"]
elixirc:
  <<: *defaults
  entrypoint: ["elixirc"]
iex:
  <<: *defaults
  entrypoint: ["iex"]
bash:
  <<: *defaults
  entrypoint: ["bash"]
```

### バージョンの確認

docker-compose.ymlに定義したbashサービスを実行してインストールしたパッケージのバージョンを確認します。

```bash
$ docker-compose run --rm bash
```

Erlang OTPのバージョンは17.5です。

```bash
$ cat /usr/lib/erlang/releases/17/OTP_VERSION
17.5
```

Elixirのバージョンは1.0.4です。

```bash
$ elixir -v
Elixir 1.0.4
```

erlはErlangのインタラクティブシェルです。バージョンは6.4です。

```bash
$ erl -version
Erlang (ASYNC_THREADS) (BEAM) emulator version 6.4
```

## iEx

Elixirでは[iEx](http://elixir-lang.org/docs/v1.0/iex/IEx.html)という専用のインタラクティブシェルが使えます。docker-compose.ymlに定義したiexサービスを起動します。

``` bash
$ docker-compose run --rm iex
Erlang/OTP 17 [erts-6.4] [source] [64-bit] [async-threads:10] [kernel-poll:false]

Interactive Elixir (1.0.4) - press Ctrl+C to exit (type h() ENTER for help)
iex(1)>
```

`Ctrl+C` を押して、a、enterで終了します。

```bash
BREAK: (a)bort (c)ontinue (p)roc info (i)nfo (l)oaded
       (v)ersion (k)ill (D)b-tables (d)istribution
```

## 文字列

[Getting Started](http://elixir-lang.org/getting-started/introduction.html)にあるサンプルを実行してみます。

```erl
iex(1)> 40 + 2
42
iex(2)> "helo" <> "world"
"heloworld"
```

`<>`は文字列結合演算子なので、2つ目の`"helo" <> "world"`は文字列の結合、またはバイナリの結合を表しています。Elixirの文字列にはダブルクォートで囲うバイナリのStringと、シングルクォートで囲うリストの文字リスト (Char list)があります。[Binaries, strings and char lists](http://elixir-lang.org/getting-started/binaries-strings-and-char-lists.html)の説明を読んでみます。

### 文字列 (String)

Elixirの文字列はUnicodeのコードポイントをUTF-8でエンコードしたバイト列です。コードポイントは整数値 (integer)です。リテラル表記はダブルクォートで囲みます。`is_binary`でtrueが返るように、文字列はバイナリです。

```erl
iex(1)> string = "hello"
"hello"
iex(2)> is_binary string
true
```

`あ`のUnicodeのコードポイントは16進数で`3042`、10進数で`12345`です。1バイトは10進数で`0`から`255`までしか表現できません。1バイトでは表現しきれないため3バイトを使います。文字列に`?`を使うとUnicodeのコードポイントを出力できます。

```erl
iex(3)> ?あ
12354
iex(4)> byte_size "あ"
3
```

また、Elixirではバイナリを`<<>>`で表記します。バイナリは`<<0,1>>`のようなバイト列です。`<>`でバイナリを結合することができます。

```erl
iex(5)> <<0,1>> <> <<2,3>>
<<0, 1, 2, 3>>
```

文字列とヌルバイトの`<<0>>`を`<>`で結合すると、最初の文字列のバイナリ表記を出力できます。

```erl
iex(6)> "あ" <> <<0>>
<<227, 129, 130, 0>>
```

`あ`のバイト列は`<<227,129,139>>`になります。

```erl
iex(7)> <<227, 129, 130>>
"あ"
```

### 文字リスト (Char list)

文字をシングルクォートで囲うと文字のリストになります。

```erl
iex(1)> char_list = 'hello'
'hello'
iex(2)> is_list char_list
true
```

文字リストはUnicodeのコードポイントで構成されるリストです。

```erl
iex(2)> 'あ'
[12354]
iex(3)> 'あい'
[12354, 12356]
iex(4)> to_char_list "あい"
[12354, 12356]
```

1バイトで表現できる文字の場合、文字リストを評価するとコードポイントでなく文字そのものが表示されます。

```erl
iex(4)> 'ab'
'ab'
```

[Binaries, strings and char lists](http://elixir-ja.sena-net.works/getting_started/6.html)によると、文字リストはバイナリを引数に取れない古いErlangのライブラリで使われているようです。

