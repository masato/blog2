title: "Windows7にTDM-GCCとMinGW-w64で64bitのPython2.7環境を構築する"
date: 2015-02-24 20:11:22
tags:
 - Windows7
 - TDM-GCC
 - MinGW-w64
 - MinGW
 - MSYS
 - Python
 - PyCUDA
 - Theano
description: PyCUDAとTheanoでGPU計算をするのが目的です。どう考えてもWindowsで64bitのPythonの開発はすべきではないのですが、手持ちのノートPCに開発環境を構築してみました。中古で古いGeForceのグラフィックボードを搭載したノートPCを購入してUbuntuをインストールした方が幸せになれます。主に以下のサイトを参考にして作業します。Installing Theano with GPU on Windows 64-bit Setting up a MinGW-w64 build environment
---

PyCUDAとTheanoでGPU計算をするのが目的です。どう考えてもWindowsで64bitのPythonの開発はすべきではないのですが、手持ちのノートPCに開発環境を構築してみました。中古で古いGeForceのグラフィックボードを搭載したノートPCを購入してUbuntuをインストールした方が幸せになれます。主に以下のサイトを参考にして作業します。

* [Installing Theano with GPU on Windows 64-bit](http://pavel.surmenok.com/2014/05/31/installing-theano-with-gpu-on-windows-64-bit/)
* [Setting up a MinGW-w64 build environment](http://ascend4.org/Setting_up_a_MinGW-w64_build_environment)

<!-- more -->

## WindowsのGCC環境

WindowsでGCCを使う場合はいろいろな方法がありますが今回は以下の組み合わせにしました。

* [MinGW](http://www.mingw.org/): Windows用のミニマルなGNU/GCC環境
* [MinGW-w64](http://mingw-w64.sourceforge.net/): MinGWからフォークした64bit環境
* [TDM-GCC](http://tdm-gcc.tdragon.net/): 最新のGCCに対応、MinGWとMinGW-w64をオプションで指定可能
* [MSYS](http://www.mingw.org/wiki/msys): コマンドライン環境

## Python 2.7

[Python 2.7 Release](https://www.python.org/download/releases/2.7/)から[python-2.7.amd64.msi](https://www.python.org/ftp/python/2.7/python-2.7.amd64.msi)のインストーラーをダウンロードして実行します。環境変数PATHにフォルダを追加します。

* コンピューター右クリック > プロパティ > システムの詳細設定 > 詳細設定 > 環境変数 > システム環境変数 > PATHの最後に追加

``` dos
C:\Python27;C:\Python27\Scripts
```

## MinGWとMSYS

[MinGW](http://www.mingw.org/)から[mingw-get-setup.exe](http://sourceforge.net/projects/mingw/files/Installer/mingw-get-setup.exe/download)のインストーラーをダウンロードして実行します。32bitと64bitのGCC環境を用意するためフォルダを分けてインストールします。

* インストール先: `c:\MinGW\32`
* graphical user interfceのチェックを外す

インストールが終了したらprofile.xmlを編集してフォルダ構成を変更します。

``` xml c:\mingw\32\var\lib\mingw-get\data\profile.xml
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<profile project="MinGW" application="mingw-get">
  <repository uri="http://prdownloads.sourceforge.net/mingw/%F.xml.lzma?download"/>
  <system-map id="default">
    <sysroot subsystem="mingw32" path="%R" />
    <sysroot subsystem="MSYS" path="%R/../msys" />
  </system-map>
</profile>
```

### mingw-getコマンドで32bit環境構築

fstabを編集して32bit環境に切り換えます。

``` bash c:\mingw\msys\etc\fstab
c:/mingw/32   /mingw
```

mingw-getコマンドを使い必要なパッケージをインストールします。

``` bash
$ mingw-get install msys-core msys-base msys-vim msys-wget msys-patch msys-flex msys-bison msys-unzip
```

32bitのgcc環境をインストールします。

``` bash
$ mingw-get install gcc g++ gfortran
```

fstabを編集してデフォルトは64bit環境を使うように設定設定しておきます。

``` bash c:\mingw\msys\etc\fstab
c:/mingw/64   /mingw
```

### msys.batのショートカット作成

デスクトップにmysyを起動するショートカットを作成します。

* デスクトップ右クリック > 新規作成 > ショートカット
* 項目の場所: `c:\mingw\msys\msys.bat`

アイコンを変更します。

* デスクトップのmsys.bat右クリック > アイコンの変更
* アイコンの場所: `c:\mingw\msys\msys.ico`


## TDM-GCC と MinGW-w64

[TDM-GCC](http://tdm-gcc.tdragon.net/)の[ダウンロードページ](http://tdm-gcc.tdragon.net/download)からTDM64 MinGW-w64 editionの[tdm-gcc-4.9.2.exe](http://sourceforge.net/projects/tdm-gcc/files/TDM-GCC%20Installer/tdm-gcc-4.9.2.exe/download)をダウンロードします。

* インストール先: `c:\mingw\64`
* 全てのcomponentsをインストール、Add to PATHはチェックを外す

システム環境変数PATHを確認して`c:\mingw\64\bin`が入っていたら削除します。

* コンピューター右クリック > プロパティ > システムの詳細設定 > 詳細設定 > 環境変数 > システム環境変数 > PATH

デスクトップのmsys.batをダブルクリックして、gccのバージョンを確認します。
  
``` bash
$ gcc --version
gcc.exe (tdm64-1) 4.9.2
Copyright (C) 2014 Free Software Foundation, Inc.
This is free software; see the source for copying conditions.  There is NO
warranty; not even for MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.
```

TDM-GCCは32bitと64bitをオプションを指定してターゲットを変更することができます。MSYSホームディレクトリの`c:\mingw\msys\home\masato\.profile`を編集してMSYSの起動時にビルド環境を表示するようにします。

``` bash ~/.profile
#!/bin/bash
# we use the reported architecture of the 'gcc' in our path to 
# determine which Python and other utilities we will be using.
_arch=`gcc -dumpmachine`
#echo "ARCH=$_arch"
if grep "^x86_64-" <<< "$_arch" >/dev/null ; then
	echo "MINGW 64 BIT BUILD ENVIRONMENT"
	_pydir="/c/Python27"
else
	echo "MINGW 32 BIT BUILD ENVIRONMENT"
	_pydir="/c/Python27_32"
fi

export PATH=$PATH:$_pydir:$_pydir/Scripts

# note that mingw-get will still install all its stuff to c:\mingw\32,
# because of the contents of its profile.xml file.
alias mingw-get="/c/mingw/32/bin/mingw-get"
```

### Fortran

[SciPy](http://www.scipy.org/)のビルドに必要なようなのでインストールしました。実際にはカスタムのインストラーを使うことにしたので不要かも知れません。TDM-GCCではデフォルトでFortranがインストールされません。

[tdm-gcc-webdl.exe](http://sourceforge.net/projects/tdm-gcc/files/TDM-GCC%20Installer/tdm-gcc-webdl.exe/download)をダウンロードします。`Manage`ボタンを押してインストールされた環境の構成変更をします。

* インストール先: `c:\mingw\64`
* 全てのcomponentsをインストール、Add to PATHはチェックを外す

## SCons

[SCons](http://www.scons.org/)のビルドツールをインストールします。デスクトップに作成したmysy.batを起動します。

``` bash
$ cd ~
$ wget http://prdownloads.sourceforge.net/scons/scons-2.3.4.tar.gz
$ tar zxf scons-2.3.4.tar.gz
$ cd scons-2.3.4
$ python setup.py bdist_wininst
```

GUIインストーラーを起動します。

``` bash
$ start dist/scons-2.3.4.win-amd64.exe 
```

Scriptsフォルダにコピーします。

``` bash
$ cp /c/Python27/Scripts/scons.py /c/Python27/Scripts/scons
```

バージョンを確認します。

``` bash
$ scons --version
SCons by Steven Knight et al.:
        script: v2.3.4, 2014/09/27 12:51:43, by garyo on lubuntu
        engine: v2.3.4, 2014/09/27 12:51:43, by garyo on lubuntu
        engine path: ['c:\\Python27\\Lib\\site-packages\\scons-2.3.4\\SCons']
Copyright (c) 2001 - 2014 The SCons Foundation
```


## Dependency Walker

[Dependency Walker](http://www.dependencywalker.com/)をインストールします。Dependency WalkerはWindowsモジュールの依存関係を調べることができます。zipをダウンロードして`/c/mingw/64/bin`に解凍します。

``` bash
$ wget http://www.dependencywalker.com/depends22_x64.zip
$ unzip depends22_x64.zip -d /c/mingw/64/bin
Archive:  depends22_x64.zip
  inflating: /c/mingw/64/bin/depends.chm
  inflating: /c/mingw/64/bin/depends.dll
  inflating: /c/mingw/64/bin/depends.exe
```

## SWIG

[SWIG](http://www.swig.org/)はC/C++のライブラリをいろいろ言語から呼び出すためのツールです。zipをダウンロードして`/c/mingw/msys/opt`にインストールします。

``` bash
$ wget http://prdownloads.sourceforge.net/swig/swigwin-3.0.5.zip
$ mkdir -p /c/mingw/msys/opt
$ unzip swigwin-3.0.5.zip -d /c/mingw/msys/opt
```

`~/.profile`を編集しPATHに追加します。

``` bash ~/.profile
export PATH=$PATH:/opt/swigwin-3.0.5
```

msys.batを再起動してバージョンを確認します。

``` bash
$ swig -version

SWIG Version 3.0.5

Compiled with i586-mingw32msvc-g++ [i586-pc-mingw32msvc]

Configured options: +pcre

Please see http://www.swig.org for reporting bugs and further information
```


## GTK+

[GTK+](http://www.gtk.org/)のGUIツールキットをインストールします。インストーラーをダウンロードして実行します。

``` bash
$ wget http://sourceforge.net/projects/ascend-sim/files/thirdparty/gtk%2B-2.22.1-20101229-x64-a4.exe/download
$ start gtk+-2.22.1-20101229-x64-a4.exe
```

`~/.profile`を編集しPATHに追加します。

``` bash ~/.profile
export PATH=$PATH:/c/Program\ Files/GTK+-2.22/bin/
```

msys.batを再起動してインストールの確認をします。gtk-domoを実行するとGUIが起動します。

``` bash
$ gtk-demo
```

## Subversion

Subversionは[SlikSVN](http://www.sliksvn.com/en/download)から[Slik-Subversion-1.8.11-x64.msi](https://sliksvn.com/pub/Slik-Subversion-1.8.11-x64.msi)ダウンロードしてインストールします。


`~/.profile`を編集しPATHに追加します。

``` bash ~/.profile
export PATH=$PATH:/c/Program\ Files/SlikSvn/bin
```

msys.batを再起動してバージョンの確認をします。

``` bash
$  svn --version
svn, version 1.8.11-SlikSvn-1.8.11-X64 (SlikSvn/1.8.11) X64
   compiled Dec  9 2014, 13:44:31 on x86_64-microsoft-windows6.2.9200

Copyright (C) 2014 The Apache Software Foundation.
```

## python27.libの再ビルド

64bit環境のPython extensionsをビルドする場合に必要な準備をします。python27.libがMinGW-w64だと動作しないため、[gendef](http://sourceforge.net/p/mingw-w64/wiki2/gendef/)を使ってビルドし直します。

``` bash
$ svn co  svn://svn.code.sf.net/p/mingw-w64/code/trunk/mingw-w64-tools/gendef -r5774 ~/gendef
$ cd ~/gendef 
$ ./configure --prefix=/mingw
$ make -j4 && make install
$ cd
$ gendef --help
Usage: gendef [OPTION]... [DLL]...
Dumps DLL exports information from PE32/PE32+ executables
...
```

エクスプローラーを開きpython27.dllをコピーします。

* コピー元: `c:\windows\system32\python27.dll`
* コピー先: `c:\Python27\libs\python27.dll`

[Creating a 64 bit development environment with MinGW on Windows](https://github.com/kivy/kivy/wiki/Creating-a-64-bit-development-environment-with-MinGW-on-Windows)を参考にしてpython27.libをビルドします。デスクトップのmysy.batショートカットを起動します。

``` bash
$ cd /c/Python27/libs
$ mv python27.lib old_python27.lib
$ gendef python27.dll
 * [python27.dll] Found PE+ image
$ dlltool --dllname python27.dll --def python27.def --output-lib python27.lib
```

`c:\Python27\include\pyconfig.h`を開き、141行目から以下3行をカットします。

``` c c:\Python27\include\pyconfig.h
#ifdef _WIN64
#define MS_WIN64
#endif
```

107行目の`#ifdef _MSC_VER`の上にカットした3行をペーストします。

``` c c:\Python27\include\pyconfig.h
#ifdef _WIN64
#define MS_WIN64
#endif
#ifdef _MSC_VER
```

## ~/.profile

最終的に以下のような`~/.profile`をMSYSのホームディレクトリ(`c:\mingw\msys\home\masato`)に配置します。

``` bash ~/.profile
#!/bin/bash
# we use the reported architecture of the 'gcc' in our path to
# determine which Python and other utilities we will be using.
_arch=`gcc -dumpmachine`
#echo "ARCH=$_arch"
if grep "^x86_64-" <<< "$_arch" >/dev/null ; then
        echo "MINGW 64 BIT BUILD ENVIRONMENT"
        _pydir="/c/Python27"
        export PATH=$PATH:/c/Program\ Files/GTK+-2.22/bin/
else
        echo "MINGW 32 BIT BUILD ENVIRONMENT"
        _pydir="/c/Python27_32"
        export PATH=$PATH:/c/Program\ Files\ \(x86\)/GTK+-2.22/bin/
fi

export PATH=$PATH:$_pydir:$_pydir/Scripts

# note that mingw-get will still install all its stuff to c:\mingw\32 and c:\min
gw\msys.
# because of the contents of its profile.xml file. it is not affected by the con
tent of /etc/fstab.
alias mingw-get="/c/mingw/32/bin/mingw-get"

export PATH=$PATH:/opt/swigwin-3.0.5
export PATH=$PATH:/c/Program\ Files/SlikSvn/bin
```