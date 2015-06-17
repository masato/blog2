title: "Elixir on Dockerで関数型言語をはじめよう - Part2: 関数の実行"
date: 2015-06-10 13:35:35
tags:
 - Elixir
 - DockerCompose
description: 前回はiExのインタラクティブシェルに実行しました。はじめてRubyでコードを書いたときのような楽しさがあります。次はファイルに書いたモジュールをコンパイルして実行してみます。またEmacsのelixir-modeをインストールして開発環境を整備していきます。
---

前回はiexのインタラクティブシェルに実行しました。はじめてRubyでコードを書いたときのような楽しさがあります。次はファイルに書いたモジュールをコンパイルして実行してみます。またEmacsのelixir-modeをインストールして開発環境を整備していきます。

## 開発環境

### Docker Compose

前回と同様にElixirのコマンドはDocker Composeで実行します。

```yml ~/elixir_apps/docker-compose.yml
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

以下のようにiexサービスを使い捨てのコンテナで起動します。

```bash
$ docker-compose run --rm iex
Erlang/OTP 17 [erts-6.4] [source] [64-bit] [async-threads:10] [kernel-poll:false]

Interactive Elixir (1.0.4) - press Ctrl+C to exit (type h() ENTER for help)
iex(1)>
```

### elixir-mode

少しずつElixirの開発環境を用意していきます。Docker Composeからカレントディレクトリをボリュームとしてコンテナの`/app`ディレクトリにマップしています。DockerホストのEmacsでElixirのファイルを編集してコンテナで実行します。

[emacs-elixir](https://github.com/elixir-lang/emacs-elixir)を[Cask](https://github.com/cask/cask)を使いEmacsにインストールします。

```bash
$ cd ~/.emacs.d
$ cat << EOF >> Cask
;; elixir
(depends-on "elixir-mode")
EOF
$ cask install
```

## コードの実行

Elixirはコードを実行するコマンドが3つあります。

* iex: インタラクティブシェルで実行
* elixirc: コンパイルして実行 (.ex)
* elixir: スクリプト実行 (.exs)


### iexでインタラクティブ実行

ErlangやScalaなどと同様でElixirは純粋に関数だけを定義することができません。

```ex
$ iex
iex(1)> def hello do
...(1)>   IO.puts "Hello World"
...(1)> end
** (ArgumentError) cannot invoke def/2 outside module
    (elixir) lib/kernel.ex:3563: Kernel.assert_module_scope/3
    (elixir) lib/kernel.ex:2823: Kernel.define/4
    (elixir) expanding macro: Kernel.def/2
             iex:1: (file)
iex(1)>
```

関数はモジュールの内部に定義する必要があります。

```ex
iex(2)> defmodule MyModule do
...(2)>   def hello do
...(2)>     IO.puts "Another Hello"
...(2)>   end
...(2)> end
```

関数の実行は`()`を省略することができます。この例のように引数がない場合は省略しても良いですが、引数がある関数の実行は`()`を付けた方が読みやすくなります。

```ex
iex(3)> MyModule.hello
Another Hello
:ok
```

hello関数をMyModuleに定義して`my_module.ex`ファイルを作成します。

```ex my_module.ex
defmodule MyModule do
  def hello do
     IO.puts "Another Hello"
  end
end
```

iexを起動してからモジュールをコンパイルとロードをします。

```ex
$ iex
iex(1)> c("my_module.ex")
[MyModule]
iex(2)> MyModule.hello
Another Hello
:ok
```

iexコマンドはモジュールが定義されたファイルを引数にするとコンパイルとロードができます。また、カレントディレクトリにコンパイル済の`.beam`ファイルがある場合は自動的にロードされます。以下のようにiexコマンドで同じ名前のモジュールを再度コンパイルした警告がでますが、この場合は無視できます。

```ex
$ ls Elixir.MyModule.beam
Elixir.MyModule.beam
$ iex my_module.ex
my_module.ex:1: warning: redefining module MyModule
iex(1)> MyModule.hello
Another Hello
:ok
```

### elixircでコンパイル

elixircコマンドでモジュールファイルをコンパイルして、`Elixir.モジュール名.beam`の中間ファイルを作成します。

```bash
$ elixirc my_module.ex
$ ls Elixir.MyModule.beam
Elixir.MyModule.beam
```

コンパイルした`.beam`ファイルのモジュールは、先ほどのようにiexやmixコマンドでロードすることができます。

### elixirでスクリプト実行

`ex`と`exs`は拡張子が違っても動作に影響はありません。スクリプトの場合`.exs`と拡張子を付けるようです。`my_module.exs`はファイル内で定義したモジュールの関数を実行しています。

```ex my_module.exs
defmodule MyModule do
  def hello do
     IO.puts "Hello Script"
  end
end

MyModule.hello
```

`elixir`コマンドで実行すると`.beam`中間ファイルを生成せずにコンパイルして実行します。

```bash
$ elixir my_module.exs
Hello Script
```
