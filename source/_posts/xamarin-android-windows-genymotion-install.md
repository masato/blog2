title: "Xamarin StudioとGenymotionでAndroid開発 - Part1: インストール"
date: 2015-01-11 20:59:58
tags:
 - Xamarin
 - XamarinStudio
 - XamarinAndroid
 - Android
 - Genymotion
description: Xamarin.Androidを開発するための環境をWindows7に用意します。続きはOSXでも開発をしたいのでIDEはXamarin Studioを単体で使うことにします。AndroidエミュレータはGenymotionを使います。Xamarinアカウントは無料のSTARTERプランではサブスクリプションがないので、Xamarin Android Playerは使えません。INDIE以上の有料サブスクリプションが必要になります。
---

Xamarin.Androidを開発するための環境をWindows7に用意します。続きはOSXでも開発をしたいのでIDEはXamarin Studioを単体で使うことにします。Androidエミュレータは[Genymotion](https://www.genymotion.com)を使います。Xamarinアカウントは無料のSTARTERプランではサブスクリプションがないので、[Xamarin Android Player]( http://developer.xamarin.com/guides/android/getting_started/installation/android-player/)は使えません。INDIE以上の有料サブスクリプションが必要になります。

<!-- more -->

### Xamarin Android Player

無料のSTARTERプランではXamarin Android Playerが使えません。OpenGLに対応しているなど高機能なので使いたいのですが、学習段階では無料プランで試したいです。他のAndroidエミュレータを探しているとGenymotionがよさそうです。

### Genymotion

[Genymotion](https://www.genymotion.com)はVirtualBox上で動作します。あらかじめVirtualBoxの最新バージョンをインストールしておきます。今回は4.3.20を使います。Genymotionのサイトで無料アカウントを作成してからインストーラーをダウンロードします。VirtualBoxはインストール済みなので、`Get Genymotion (without VirtualBox) (25.39MB)`をダウンロードします。


Genymotionのバージョンは2.3.1です。

{% img center /2015/01/11/xamarin-android-windows-genymotion-install/genymotion-version.png %}


### Google Nexus 4 - 4.4.4 - (API19) - 768x1280

Genymotionを起動すると初回に`Add a first virtual device`のダイアログが表示されます。

{% img center /2015/01/11/xamarin-android-windows-genymotion-install/genymotion-add-first-device.png %}

VirtualBoxが起動していない状態で、Genymotionを起動します。ログインして`Availavle virtual devices`を表示します。`Google Nexus 4 - 4.4.4 - (API19) - 768x1280`のvirtual deviceを追加します。

{% img center /2015/01/11/xamarin-android-windows-genymotion-install/genymotion-list.png %}

`Your virtual devices`からインストールしたデバイスを選択して起動します。

{% img center /2015/01/11/xamarin-android-windows-genymotion-install/genymotion-nexus4-virtual-device.png %}

### Xamarinアカウント

[Xamarin](http://xamarin.com/)のサイトからアカウントを作成します。最初なので無料のSTARTERプランのまま続けます。

### Xamarinのインストール

[Download](http://xamarin.com/download)ページからXamarinInstaller.exeをダウンロードして実行します。Requirementsも自動でダウンロードしてくれます。

* Java JDK 1.7
* Android SDK 22.0.0
* GTK# 2.12.25
* Xamarin Studio 5.5.4
* Xamarin 3.8.150

環境に依存しますが1時間くらいすべてインストールにかかりました。Xamarin Studioを起動して右上の`Log in`からXamarinアカウントにログインします。

{% img center /2015/01/11/xamarin-android-windows-genymotion-install/xamarin-studio-windows-login.png %}

Xamarin Studioのバージョンは5.5.4です。

{% img center /2015/01/11/xamarin-android-windows-genymotion-install/xamarin-studio-version.png %}










