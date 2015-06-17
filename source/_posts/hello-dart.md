title: 'Ubuntu 14.04のEmacs24でDart開発環境'
date: 2014-05-18 14:50:11
tags:
 - Dart
 - Emacs
 - RikuloStream
 - Ubuntu
description: Ubuntu 14.04のDockerコンテナにEmacsでDartの開発ができるように準備していきます。以前使ったEmacsの設定が基本になります。
---
Ubuntu 14.04のDockerコンテナにEmacsでDartの開発ができるように準備していきます。
[以前](/2014/04/27/emacs-init/)使ったEmacsの設定が基本になります。

<!-- more -->

### Dockerコンテナの起動

[前回](/2014/05/18/docker-dart-redstone/)`Dart SDK`をインストールしたイメージを使います。
Webアプリ用に8080をポートフォワードします。

``` bash
$ docker run -d -p 8080:8080  masato/dart-sdk:latest /sbin/my_init --enable-insecure-key
$ docker ps
CONTAINER ID        IMAGE                    COMMAND                CREATED             STATUS              PORTS                    NAMES
64f66f89b0c3        masato/dart-sdk:latest   /sbin/my_init --enab   2 hours ago         Up 2 seconds        0.0.0.0:8080->8080/tcp   grave_turing
```

### Emacs24のインストール

コンテナのUbuntuのバージョンを確認します。

``` bash
# cat /etc/lsb-release
DISTRIB_ID=Ubuntu
DISTRIB_RELEASE=14.04
DISTRIB_CODENAME=trusty
DISTRIB_DESCRIPTION="Ubuntu 14.04 LTS"
```

Emacsのインストール

``` bash
# apt-get update
# apt-get install emacs24-nox emacs24-el 
# emacs -version
GNU Emacs 24.3.1
```

### dart-mode
[dart-mode](https://github.com/milkypostman/melpa/blob/master/recipes/dart-mode)は[MELPA](http://melpa.milkbox.net/)からダウンロードするので、package-user-dirにリポジトリを追加します。
[Marmalade](http://marmalade-repo.org/)も一緒に追加しておきます。

``` el ~/.emacs.d/init.el
;; package
(require 'package)
(add-to-list 'package-archives '("melpa" . "http://melpa.milkbox.net/packages/") t)
(add-to-list 'package-archives '("marmalade" . "http://marmalade-repo.org/packages/"))
(package-initialize)
```

Emacsを起動し、`M-x list-packages`をするとパッケージ一覧が表示されます。
リストが表示されたら、dart-modeを選択(i)して、実行(x)を押しインストールします。
また、削除したい場合は削除する場合は、選択(d)と実行(x)をします。

### Dart 1.3.6

Dartのバージョンを確認します。

``` bash
# which dart
/root/dart-sdk/bin/dart
# dart --version
Dart VM version: 1.3.6 (Tue Apr 29 12:40:24 2014) on "linux_x64"
```

### Rikulo StreamでHello Wold

[Hello Wold](http://docs.rikulo.org/stream/latest/Getting_Started/Hello_World.html)を写経しながら、かんたんあWebアプリを[Rikulo Stream](http://rikulo.org/projects/stream)で書いてみます。
`Rikulo Stream`はシンプルなMVCフレームワークです。まずはプロジェクトの作成から。

``` bash
# mkdir -p ~/dart_apps/hello-world/web/webapp
# cd ~/dart_apps/hello-world
```

pubspec.yamlを書きます。

``` yaml ~/dart_apps/hello-world/pubspec.yaml
name: helloworld
version: 1.0.0
description: Hello World Sample Application
dependencies:
  stream: any
```
pub getでパッケージをダウンロードします。

``` bash
# pub get
Resolving dependencies... (7.9s)
Downloading stream 1.2.0...
Downloading path 1.1.0...
Downloading args 0.11.0+1...
Downloading rikulo_commons 1.0.4...
Downloading collection 0.9.2...
Downloading mime 0.9.0+1...
Got dependencies!
```

main.dartを書きます。

``` dart ~/dart_apps/hello-world/web/webapp/main.dart
import "package:stream/stream.dart";

void main() {
  new StreamServer().start();
}
```

index.htmlを書きます。

``` html ~/dart_apps/hello-world/web/index.html
<!DOCTYPE html>
<html>
  <head>
    <title>Hello World!</title>
  </head>
  <body>
    <h1>Welcome to Stream!</h1>
  </body>
</html>
```

作成したWebアプリのディレクトリ構造は以下になります。

```
hello-world/
  pubspec.yaml
  web/
    index.html
    webapp/
      main.dart
```

[multiterm](http://www.emacswiki.org/emacs/MultiTerm)を起動して、dartプログラムを起動します。
`C-x 3`で、ウィンドウを分割、`C-x o`で分割したウィンドウに移動します。
`M-x multi-term`でEmacs上にシェルを開きます。

``` bash
# dart web/webapp/main.dart
2014-05-18 05:39:29.226:stream:0
INFO: Rikulo Stream Server 1.2.0 starting on 0.0.0.0:8080
Home: /root/dart_apps/hello-world/web
```

### Dockerホストから確認

Dockerホストから確認します。

```
# curl localhost:8080
<!DOCTYPE html>
<html>
  <head>
    <title>Hello World!</title>
  </head>
  <body>
    <h1>Welcome to Stream!</h1>
  </body>
</html>
```
### まとめ
Dartは始めたばかりなので、サーバーサイドのフレームワークが何がよいかいろいろと試してみようと思います。
開発環境に依存したくないので、なるべくDart Editorを使わずにエディタでプログラムが書ける環境が欲しいところです。

multi-termを使うとDockerコンテナの中でEmacsをIDEとして利用できます。
また、どうやらNitrous.IOでもDartが[書ける](https://github.com/nitrous-io/autoparts/blob/master/lib/autoparts/packages/dart.rb)らしいです。
