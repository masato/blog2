title: "Xamarin StudioとGenymotionでAndroid開発 - Part2: Hello, Android"
date: 2015-01-12 00:13:58
tags:
 - Xamarin
 - XamarinStudio
 - XamarinAndroid
 - Android
 - Genymotion
description: Windows7にインストールしたXamarinStudioを使ってXamarin.Androidのサンプルを実装します。Hello, Androidのチュートリアルを使います。手順ではプロジェクト名がPhoneword_Droidですが、Phoneword.Androidに変更してnamespaceにも使います。完成したコードはGitHubのPhoneWordのリポジトリにあります。
---

[Windows7にインストールした](/2015/01/11/xamarin-android-windows-genymotion-install/)XamarinStudioを使ってXamarin.Androidのサンプルを実装します。[Hello, Android](http://developer.xamarin.com/guides/android/getting_started/hello,android/hello,android_quickstart/)のチュートリアルを使います。手順ではプロジェクト名が`Phoneword_Droid`ですが、`Phoneword.Android`に変更してnamespaceにも使います。完成したコードはGitHubの[PhoneWord](https://github.com/masato/PhoneWord)のリポジトリにあります。

<!-- more -->

### Getting Started

Xamarin DevelopersのGetting Startedを開きます。

* [Getting Started with Android](http://developer.xamarin.com/guides/android/getting_started/)
* [Hello, Android](http://developer.xamarin.com/guides/android/getting_started/hello,android/)
* [Part 1: Quickstart](http://developer.xamarin.com/guides/android/getting_started/hello,android/hello,android_quickstart/)

### Xamarin App Icons & Launch Screens set

プロジェクトに追加するアインコンの[Xamarin App Icons & Launch Screens set](http://developer.xamarin.com/guides/android/getting_started/hello,android/Resources/XamarinAppIconsAndLaunchImages.zip)のzipファイルをダウンロードして解凍します。


### PhoneWordソリューションの作成

PhoneWordソリューションを作成します。

* New Solution... -> その他 > 空のソリューション

{% img center /2015/01/12/xamarin-android-windows-genymotion-hello-android/xamarin-studio-solution.png %}


### PhoneWord.Androidプロジェクトの作成

新しいプロジェクトのダイアログを表示します。

* PhoneWordソリューションのギアアインコン -> 追加 -> 新しいプロジェクトを追加

PhoneWord.Androidプロジェクトを作成します。

* C# -> Android -> Android Applicationテンプレート

{% img center /2015/01/12/xamarin-android-windows-genymotion-hello-android/xamarin-studio-project.png %}

### Main.axmlの編集

デザイン サーフェイスを開きます。

* Resourcesフォルダ -> layoutフォルダ ->  Main.axmlをダブルクリック

Buttonを選択して、Deleteキーを押して削除します。

Toolboxから、Text (Large) コントロールを、デザイン サーフェイスへドラッグ＆ドロップします。Text (Large) コントロールを選択して、プロパティ パッドでTextプロパティを編集します。

* Text property: Enter a Phoneword:

Plain Text コントロールを、 デザイン サーフェイスへ、Text (Large)コントロールの下にドラッグ＆ドロップします。

* Plain Text コントロールを選択し、Id プロパティとTextプロパティ変更する
* Id property: @+id/PhoneNumberText
* Text property: 1-855-XAMARIN

Toolboxから、Button コントロールを、デザイン サーフェイスへ、Plain Text コントロールの下にドラッグ＆ドロップします。

* Buttonコントロールを選択して、 プロパティ パッドから、Id プロパティとTextプロパティ変更する
* Id property: @+id/TranslateButton
* Text property: Translate


Toolboxから、Button コントロールを、デザイン サーフェイスへ、TranslateButtonの下にドラッグ＆ドロップします。

* Buttonコントロールを選択して、 プロパティ パッドから、Id プロパティとTextプロパティ変更する
* Id property: @+id/CallButton
* Text property: Call

ここまでの作業を`Ctrl + s`で保存します。

{% img center /2015/01/12/xamarin-android-windows-genymotion-hello-android/xamarin-studio-Main.png %}

### PhoneTranslator.cs の追加

PhoneTranslator.csを追加して、電話番号を英数字から数字に変換するコードを記述します。

* PhoneWordソリューション  -> PhoneWord.Android プロジェクト
* ギアアインコン -> 追加 -> General -> 新しいファイルを追加
* General -> 空のクラス
* 名前: PhoneTranslator

{% img center /2015/01/12/xamarin-android-windows-genymotion-hello-android/xamarin-studio-PhoneTranslator.png %}

以下のようにPhoneTranslator.csを作成します。

``` c# C:\Users\masato\Documents\Projects\PhoneWord\PhoneWord.Android\PhoneTranslator.cs
using System.Text;
using System;

namespace PhoneWord.Android
{
    public static class PhonewordTranslator
    {
        public static string ToNumber(string raw)
        {
            if (string.IsNullOrWhiteSpace(raw))
                return "";
            else
                raw = raw.ToUpperInvariant();

            var newNumber = new StringBuilder();
            foreach (var c in raw)
            {
                if (" -0123456789".Contains(c))
                    newNumber.Append(c);
                else {
                    var result = TranslateToNumber(c);
                    if (result != null)
                        newNumber.Append(result);
                }
                // otherwise we've skipped a non-numeric char
            }
            return newNumber.ToString();
        }
        static bool Contains (this string keyString, char c)
        {
            return keyString.IndexOf(c) >= 0;
        }
        static int? TranslateToNumber(char c)
        {
            if ("ABC".Contains(c))
                return 2;
            else if ("DEF".Contains(c))
                return 3;
            else if ("GHI".Contains(c))
                return 4;
            else if ("JKL".Contains(c))
                return 5;
            else if ("MNO".Contains(c))
                return 6;
            else if ("PQRS".Contains(c))
                return 7;
            else if ("TUV".Contains(c))
                return 8;
            else if ("WXYZ".Contains(c))
                return 9;
            return null;
        }
    }
}
```

`Ctrl + s`でPhoneTranslator.csを保存します。

### MainActivity.cs

PhoneWordソリューション から、MainActivity.csをダブルクリックして、ユーザーインタフェースの記述をします。以下のような状態のOnCreateメソッドに追加してコードを書いていきます。

``` c# C:\Users\masato\Documents\Projects\PhoneWord\PhoneWord.Android\MainActivity.cs
using System;

using Android.App;
using Android.Content;
using Android.Runtime;
using Android.Views;
using Android.Widget;
using Android.OS;

namespace Phoneword.Android
{
    [Activity (Label = "Phoneword.Android", MainLauncher = true)]
    public class MainActivity : Activity
    {
        protected override void OnCreate (Bundle bundle)
        {
            base.OnCreate (bundle);

            // Set our view from the "main" layout resource
            SetContentView (Resource.Layout.Main);

        }
    }
}
```

OnCreateメソッドにコントローラーへの参照を追加します。

``` c# C:\Users\masato\Documents\Projects\PhoneWord\PhoneWord.Android\MainActivity.cs
// Get our UI controls from the loaded layout
EditText phoneNumberText = FindViewById<EditText>(Resource.Id.PhoneNumberText);
Button translateButton = FindViewById<Button>(Resource.Id.TranslateButton);
Button callButton = FindViewById<Button>(Resource.Id.CallButton);
```

TranslateButtonのイベントリスナーを追加します。

``` c# C:\Users\masato\Documents\Projects\PhoneWord\PhoneWord.Android\MainActivity.cs
// Disable the "Call" button
callButton.Enabled = false;

// Add code to translate number
string translatedNumber = string.Empty;

translateButton.Click += (object sender, EventArgs e) =>
    {
        // Translate user’s alphanumeric phone number to numeric
        //translatedNumber = Core.PhonewordTranslator.ToNumber(phoneNumberText.Text);
        translatedNumber = PhoneWord.Android.PhonewordTranslator.ToNumber(phoneNumberText.Text);

        if (String.IsNullOrWhiteSpace(translatedNumber))
        {
            callButton.Text = "Call";
            callButton.Enabled = false;
        }
        else
        {
            callButton.Text = "Call " + translatedNumber;
            callButton.Enabled = true;
        }
   };
```

CallButtonのイベントリスナーを追加しますが、namespaceを定義しているため以下のように`using Uri = Android.Net.Uri;`を追加します。

``` c# C:\Users\masato\Documents\Projects\PhoneWord\PhoneWord.Android\MainActivity.cs
using Uri = Android.Net.Uri;
...
callButton.Click += (object sender, EventArgs e) =>
{
    // On "Call" button click, try to dial phone number.
    var callDialog = new AlertDialog.Builder(this);
    callDialog.SetMessage("Call " + translatedNumber + "?");
    callDialog.SetNeutralButton("Call", delegate {
           // Create intent to dial phone
           var callIntent = new Intent(Intent.ActionCall);
           //callIntent.SetData(Android.Net.Uri.Parse("tel:" + translatedNumber));
           callIntent.SetData(Uri.Parse("tel:" + translatedNumber));
           StartActivity(callIntent);
       });
    callDialog.SetNegativeButton("Cancel", delegate { });

    // Show the alert dialog to the user and wait for response.
    callDialog.Show();
};
```

### AndroidManifest.xmlにphone callパーミッションを追加

AndroidManifest.xmlが見つからない場合は、一度ソリューションを閉じて開き直します。

* PhoneWordソリューション -> Properties -> AndroidManifest.xml
* Required Permissions -> Call Phone にチェック

{% img center /2015/01/12/xamarin-android-windows-genymotion-hello-android/xamarin-studio-CallPhone.png %}

### ビルド

ビルドに成功するとXamarin Studioの上部メッセージ欄に「ビルド成功」のメッセージが表示されます。

* メニュー -> ビルド -> すべてビルド

{% img center /2015/01/12/xamarin-android-windows-genymotion-hello-android/xamarin-studio-build.png %}

### MainActivity.csのLabel編集

スクリーンのトップに表示されるLabelを編集します。

``` c# C:\Users\masato\Documents\Projects\PhoneWord\PhoneWord.Android\MainActivity.cs
namespace PhoneWord.Android
{
    [Activity (Label = "PhoneWord", MainLauncher = true)]
    public class MainActivity : Activity
    {
        ...
    }
}
```

### AndroidManifest.xmlにアプリ名とアイコンを追加

* Application name: Phoneword

デフォルトのIcon.pngを右クリックして削除します。

* Resource -> drawble -> Icon.pngを右クリックして削除 -> Delete

ダウンロードして解凍したXamarin App Icons setのフォルダからアイコンをコピーします。

* Resource -> drawbleを右クリック -> 追加 -> ファイルを追加

ダイアログでCopy the file to the directoryを選択します。

* XamarinAppIconsAndLaunchImages -> Xamarin App Icons and Launch Images -> Android  drawable -> Icon.png

### drawable-* フォルダを追加

残りのdrawable-* フォルダのIcon.pngをすべてフォルダごとコピーします。

* Resourceを右クリック -> 追加 ->  Add Existing Folder

{% img center /2015/01/12/xamarin-android-windows-genymotion-hello-android/xamarin-studio-all-Icons.png %}

AndroidManifest.xmlにアイコンを追加します。

* AndroidManifest.xml ->  Application icon -> @drawable/iconを選択

### Genymotionを起動

一覧から`Google Nexus 4 - 4.4.4 - (API19) - 768x1280`を起動します。

{% img center /2015/01/12/xamarin-android-windows-genymotion-hello-android/genymotion-list.png %}

エミュレータが起動しました。

{% img center /2015/01/12/xamarin-android-windows-genymotion-hello-android/genymotion-nexus4-virtual-device.png %}

Xamarin Studioのデバイスの選択ボックスからGenymotionのデバイスを選択します。

* Phisical Devices -> Genymo...(API19)

Xamarin Studioの左上の再生ボタンをクリックしてDebug実行します。

{% img center /2015/01/12/xamarin-android-windows-genymotion-hello-android/genymotion-nexus4-api19.png %}


### Genymotionのエミュレータで確認

PhoneWordアイコンをタップしてアプリを起動します。

{% img center /2015/01/12/xamarin-android-windows-genymotion-hello-android/genymotion-phoneword-initial.png %}

Translateボタンを押します。Callボタンがアクティブになり、数値に変換された電話番号が表示されました。

{% img center /2015/01/12/xamarin-android-windows-genymotion-hello-android/genymotion-phoneword-translate.png %}

Callボタンを押します。電話をかけるダイアログが表示されました。

{% img center /2015/01/12/xamarin-android-windows-genymotion-hello-android/genymotion-phoneword-call.png %}


### Git

[Version Control in Xamarin Studio](http://forums.xamarin.com/discussion/13221/version-control-in-xamarin-studio)を参考にして、PhoneWordソリューションディレクトリに`.gitignore`を作成します。

``` bash C:\Users\masato\Documents\Projects\PhoneWord\.gitignore
#Autosave files
*~

# Build/test output
obj
bin
test-results

# VS sometimes writes these for no apparent reason
_UpgradeReport_Files
UpgradeLog.*

# Per-user data
*.DS_Store
*.sln.cache
*.suo
*.userprefs
*.usertasks
*.user

# ReSharper
*_Resharper.*
*.Resharper

# dotCover
*.dotCover
```

ローカルにGitのリポジトリを作成してcommitします。

``` bash
> cd C:\Users\masato\Documents\Projects\PhoneWord
> git init
> git add -A
> git commit -m "initilal commit"
```

GitHubにリモートリポジトリを作成して追加します。

``` bash
$ git remote add origin https://github.com/masato/PhoneWord.git
$ git push -u origin master
Username for 'https://github.com': masato
Password for 'https://masato@github.com':
Counting objects: 33, done.
Delta compression using up to 4 threads.
Compressing objects: 100% (24/24), done.
Writing objects: 100% (33/33), 28.82 KiB | 0 bytes/s, done.
Total 33 (delta 4), reused 0 (delta 0)
To https://github.com/masato/PhoneWord.git
 * [new branch]      master -> master
Branch master set up to track remote branch master from origin.
```
