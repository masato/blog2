title: "Xamarin StudioとGenymotionでAndroid開発 - Part3: OSX環境でGitHubからビルドする"
date: 2015-01-16 01:38:53
tags:
 - Xamarin
 - XamarinStudio
 - XamarinAndroid
 - Android
 - Genymotion
 - OSX
description: 前回まではWindowsにインストールしたXamarin Studioで作業をしていました。GitHubからソリューションをチェックアウトしてOSX上に開発環境をつくります。Windowsと同様にシュミレーターはGenymotnionを使います。
---

[前回までは](/2015/01/12/xamarin-android-windows-genymotion-hello-android/)WindowsにインストールしたXamarin Studioで作業をしていました。GitHubからソリューションをチェックアウトしてOSX上に開発環境をつくります。Windowsと同様にシュミレーターはGenymotnionを使います。

<!-- more -->

## Genymotion

Windowsと同様にVirtualBoxとGenymotionをインストールして、virtual deviceを作成します。

* VirtualBox: 4.3.20
* [Genymotion](https://www.genymotion.com): 2.3.1
* virtual device: Google Nexus 4 - 4.4.4 - (API19) - 768x1280

`Google Nexus 4 - 4.4.4 - (API19) - 768x1280`のvirtual deviceを起動します。

{% img center /2015/01/16/xamarin-android-osx-install/genymotion-osx-nexus4.png %}


## Xamarin

WindowsへのXamarinのインストールは1時間くらいかかりましたが、OSXへは5分程度で終了しました。OSXにはJavaとGTK+がインストール済みなので速いようです。

* Mono Framework: 3.12.0
* Android SDK: 22.0.0
* Xamarin Studio: 5.7.0
* Xamarin.Android: 4.20.0

{% img center /2015/01/16/xamarin-android-osx-install/osx-xamarin-version.png %}



Xamarin Studioを起動します。

{% img center /2015/01/16/xamarin-android-osx-install/xamarin-studio-osx-start.png %}

## GitHubからソリューションのチェックアウト

Projectsディレクトリの作成します。

``` bash
$ mkdir ~/Projects
```

GitHubからWindowsで開発したソリューションをcloneするため、`Version Control`の機能を使います。

* Version Control > チェックアウト > リポジトリの選択ダイアログ > 登録済みリポジトリタブ > 追加ボタン

リポジトリの設定ダイアログに、GitHubからHTTPのURLをペーストします。

{% img center /2015/01/16/xamarin-android-osx-install/phoneword-select-repository-dialog.png %}


リポジトリの選択ダイアログに戻り、ターゲットディレクトリを入力してOKボタンを押します。

* ターゲットディレクトリ: /Users/mshimizu/Projects/PhoneWord

{% img center /2015/01/16/xamarin-android-osx-install/phoneword-target-directory.png %}

GitHubのクリデンシャルを入力するダイアログが表示されるので、UsernameとPasswordを入力します。git`git clone`に成功するとPhpneWordソリューションのウィンドウが開きます。

{% img center /2015/01/16/xamarin-android-osx-install/phoneword-solution.png %}

もしUsernameやPasswordを間違えて入力してしまった場合は、ソリューションを削除してやり直す必要がありそうです。Xamarin Studioにはリポジトリ設定を編集するメニューや画面がみつまりませんでした。

## ソリューションのビルド

メニューからビルドを実行して、Xamarin Studioの左上の再生ボタンをクリックしてDebug実行します。

{% img center /2015/01/16/xamarin-android-osx-install/xamarin-osx-debug.png %}

Windowsで開発したC#のコードをOSXでビルドして、無事にアプリを起動することができました。

{% img center /2015/01/16/xamarin-android-osx-install/genymotion-osx.png %}
