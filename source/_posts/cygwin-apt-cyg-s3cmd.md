title: 'WindowsにCygwinでapt-cygとs3cmdをインストール'
date: 2014-06-07 21:14:49
tags:
 - Cygwin
 - apt-cyg
 - s3cmd
 - IDCFオブジェクトストレージ
 - Windows
description: IDCフロンティアのオブジェクトストレージはS3互換が高いので、s3cmdが使えます。OSXやUbuntuの場合は簡単に使えますが、たまにWindowsからもファイルをputしたいときがあります。WindowsでもGUIのツールの他に簡単にs3cmdが使えると便利なので、Cygwinにapt-cygでビルド環境を構築して、s3cmdをインストールしてみます。

---
IDCフロンティアのオブジェクトストレージはS3互換が高いので、s3cmdが使えます。
OSXやUbuntuの場合はs3cmdを簡単に使えますが、たまにWindowsからもファイルをputしたいときがあります。
WindowsでもGUIの他に簡単にs3cmdが使えると便利なので、[Cygwinにapt-cyg](http://code.google.com/p/apt-cyg/)でビルド環境を構築して、
s3cmdをインストールしてみます。

<!-- more -->

### Cygwinのインストール

[Cygwin](https://cygwin.com/install.html)のサイトから[setup-x86_64.exe](https://cygwin.com/setup-x86_64.exe)をダウンロードして実行します。

### apt-cygのインスト－ル

Cygwinのパッケージインストールは、setup-x86_64.exeを使うのですが、Cygwinでもapt-getみたいに、apt-cygを使います。

apt-cygのインストールに必要なパッケージは、setup-x86_64.exeからインストールします。
* Devel/git-svn
* Base/gawk 
* Utils/bzip2 
* Utils/tar 
* Web/wget

apt-cygをインストールします。

``` bash
$ svn --force export http://apt-cyg.googlecode.com/svn/trunk/ /bin/
$ chmod +x /bin/apt-cyg
```

日本のリポジトリの指定

``` bash
$ apt-cyg -m http://ftp.iij.ad.jp/pub/cygwin/ update
```

### s3cmdのインストール

s3cmdのインストールに必要な依存パッケージをインストールします。

``` bash
$ apt-cyg update
$ apt-cyg install git python python-setuptools
$ easy_install python-dateutil
```

s3cmdを`git clone`して、setup.pyします。

``` bash
$ git clone https://github.com/s3tools/s3cmd.git
$ cd s3cmd
$ python setup.py install
$ s3cmd --version
s3cmd version 1.5.0-beta1
```

### s3cmdの使い方

[以前](/2014/05/20/idcf-storage/)インストールした方法と同じです。

``` bash
$ s3cmd --configure
```

保存された.s3cfを修正します。

``` python ~/.s3cfg
access_key = {確認したAccess Key}
host_base = ds.jp-east.idcfcloud.com
host_bucket = %(bucket)s.ds.jp-east.idcfcloud.com
secret_key = {確認したSecret Key}
```

バージョンを確認します。

``` bash
$ s3cmd --version
s3cmd version 1.5.0-beta1
```

### まとめ

Cygwinを使うとUNIXコマンドを実行できますが、
今はOSXが安価に使える時代なので、積極的に`Windows + Cygwin`を選択する理由はほとんどないと思います。なんだかんだでWindows上で開発するのは面倒です。




