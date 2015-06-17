title: "Microsoft AzureにSalt環境を構築する - Part1: xplat-cliのインストール"
date: 2014-12-16 20:06:34
tags:
 - Azure
 - Salt
 - xplat-cli
 - Nodejs
description: 新しくSaltクラスタをAzureに構築してみようと思います。LinuxからAzureのコマンドラインを使うため、xplat-cli(Microsoft Azure Cross Platform Command Line)をインストールして使います。
---

新しくSaltクラスタをAzureに構築してみようと思います。LinuxからAzureのコマンドラインを使うため、xplat-cli([Microsoft Azure Cross Platform Command Line](http://azure.microsoft.com/ja-jp/documentation/articles/xplat-cli/))をインストールして使います。

<!-- more -->

## xplat-cli用のコンテナ

xplat-cliはNode.jsで書かれたツールなので、Node.js用コンテナを[dockerfile/nodejs](https://registry.hub.docker.com/u/dockerfile/nodejs/)から作成します。

``` bash
$ docker pull
$ docker run -it --name nodedev dockerfile/nodejs
```

azure-cliをインストールします。

``` bash
$ npm install -g azure-cli
```

バージョンを確認します。

``` bash
$ azure -v
0.8.13
```

## アカウント設定

アカウント設定ファイルをダウンロードするURLが表示されるので、Webブラウザで開き、Microsoft Azureにログインします。xxx-credentials.publishsettingsの名前の、アカウント設定ファイルをダウンロードします。

``` bash
$ azure account download
info:    Executing command account download
info:    Launching browser to http://go.microsoft.com/fwlink/?LinkId=254432
help:    Save the downloaded file, then execute the command
help:      account import <file>
info:    account download command OK
```

ダウンロードしたアカウント設定ファイルをインポートします。

``` bash
$ chmod 400 .publishsettings
$ azure account import .publishsettings
info:    Executing command account import
info:    account import command OK
```

## アカウント情報の確認

アカウント情報をコマンドから確認します。

``` bash
$ azure account list
info:    Executing command account list
data:    Name  Id                                    Current
data:    ----  ------------------------------------  -------
data:    従量課金  xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx  true
info:    account list command OK
```

### azure-cliの動作確認

CLIの動作確認のため、Microsoft Azureのロケーションをリストしてみます。

``` bash
$ azure vm location list
info:    Executing command vm location list
+ Getting locations
data:    Name
data:    ----------------
data:    West Europe
data:    North Europe
data:    East US 2
data:    Central US
data:    South Central US
data:    West US
data:    North Central US
data:    East US
data:    Southeast Asia
data:    East Asia
data:    Japan West
data:    Japan East
data:    Brazil South
info:    vm location list command OK
```
