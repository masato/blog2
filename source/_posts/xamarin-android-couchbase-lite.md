title: "Xamarin StudioとGenymotionでAndroid開発 - Part4: Couchbase Liteアプリを使う"
date: 2015-01-17 12:32:52
tags:
 - XamarinAndroid
 - CouchbaseLite
description: Couchbase Liteはモバイル端末向けの組み込みのJSONドキュメントデータベースです。エンジンはSQLiteです。Couchbase Mobileというモバイルソリューションでは、リモートのCouchbase ServerとJSONドキュメントを同期をすることができます。もちろん単体でSQLiteの代わりにJSONをローカルに保存する組み込みデータベースとしても使えます。Building your first Couchbase Lite .NET appのチュートリアルを写経していきます。
---

[Couchbase Lite](http://developer.couchbase.com/mobile/#couchbase-lite)はモバイル端末向けの組み込みのJSONドキュメントデータベースです。エンジンはSQLiteです。[Couchbase Mobile](http://developer.couchbase.com/mobile/)というモバイルソリューションでは、リモートのCouchbase ServerとJSONドキュメントを同期をすることができます。もちろん単体でSQLiteの代わりにJSONをローカルに保存する組み込みデータベースとしても使えます。[Building your first Couchbase Lite .NET app](http://developer.couchbase.com/mobile/develop/training/build-first-net-app/index.html)のチュートリアルを写経していきます。

<!-- more -->

## プロジェクトの作成

Xamarin Studioを起動して[プロジェクトを作成](http://developer.couchbase.com/mobile/develop/training/build-first-net-app/create-new-project/index.html)します。

* ファイル > 新規 > ソリューション

新しいソリューションダイアログから、Android Applicationを選択します。

* C# > Android > Android Application
* ソリューション名: HelloCBL

NuGetから[Couchbase Lite .NET library](https://www.nuget.org/packages/Couchbase.Lite/) をパッケージに追加します。

* プロジェクト > Add Nuget Packages...
* 右上のボックスから"Couchbase Lite"を検索する
* 最新バージョンを選択し、1.0.0をAdd Packageする


{% img center /2015/01/17/xamarin-android-couchbase-lite/couchbase_lite_add_packege.png %}


Packege Consoleに警告がでていますが、パッケージのインストールは成功したようです。

```
>> Assembly references are being added from 'lib/MonoAndroid'
Added reference 'Couchbase.Lite' to project 'HelloCBL'.
Added file 'packages.config'.
Added file 'packages.config' to project 'HelloCBL'.
Successfully added 'Couchbase.Lite 1.0.0' to HelloCBL.

Couchbase.Lite Package contains PowerShell scripts which will not be run.
```

MainActivity.csに宣言を追加します。

``` csharp:MainActivity.cs
using Couchbase.Lite;
```

## マネージャーを作成

[マネージャーを作成します](http://developer.couchbase.com/mobile/develop/training/build-first-net-app/create-database/index.html)マネージャーのインスタンスを作成します。MainActivity.csのOnCreateに実装を追加していきます。

``` csharp:MainActivity.cs
// Create a shared manager
var manager = Manager.SharedInstance;
Console.WriteLine("Manager created");
```

無料のSTARTプランを使っているのでアカウントにライセンスがついていません。ビルドに失敗してしまいます。

{% img center /2015/01/17/xamarin-android-couchbase-lite/xamarin_unavailable_features.png %}

ダイアログに表示されている`Begin a Trial`をクリックしてXamarin.Androidの30日トライアルを開始します。

{% img center /2015/01/17/xamarin-android-couchbase-lite/xamarin_indie_or_higher_required.png %}

一度全てクリーンしてからビルドし直します。Genymotionのシュミレーターを起動してからDebug実行します。

{% img center /2015/01/17/xamarin-android-couchbase-lite/xamarin_debug.png %}

アプリケーション出力にメッセージが出力されました。


```
Manager created
```

## データベースを作成

次にデータベースを作成します。

``` csharp:MainActivity.cs
// Create database
var dbName = "hello";
var database = manager.GetDatabase(dbName);
Console.WriteLine("Database created");
```

データベースの命名規則は以下です。

* Lowercase letters: a-z
* Numbers: 0-9
* Special characters: _$()+-/

Debug実行をすると、アプリケーション出力にメッセージが出力されます。

```
Database created
```

## CRUD操作を実装

[CRUD操作](http://developer.couchbase.com/mobile/develop/training/build-first-net-app/crud/index.html)をOnCreateメソッドに実装します。

JSONドキュメントのCREATEをします。

``` csharp:MainActivity.cs
using System.Collections.Generic;
...
// Create a document
var properties = new Dictionary <string,object> () {
	{"message","Hello Couchbase Lite"},
	{"created_at",DateTime.UtcNow.ToString("o")},
};

var document = database.CreateDocument ();
var revision = document.PutProperties (properties);
var docId = document.Id;
Console.WriteLine ("Document created with ID = {0}", docId);
```

Debug実行、アプリケーション出力にメッセージが出力されました。

```
Document created with ID = a6532552-c78b-4b31-b8e1-e127581493d6
```

JSONドキュメントのREADをします。

``` csharp:MainActivity.cs
...
var retrievedDocument = database.GetDocument (docId);
Console.WriteLine ("Retrieved document: ");
foreach (var kvp in retrievedDocument.Properties) {
	Console.WriteLine ("{0} : {1}", kvp.Key, kvp.Value);
}
```

Debug実行、アプリケーション出力にメッセージが出力されました。

```
message : Hello Couchbase Lite
created_at : 1/14/2015 8:51:04 AM
_id : 8c702e41-d8bb-4861-b819-bf0547ee546c
_rev : 1-5ce83ca798342e9b21ef80ae825ef7fe
```
 
JSONドキュメントのUPDATEをします。

``` csharp:MainActivity.cs
...
// Update a document
var updatedProperties = new Dictionary<string,object> (retrievedDocument.Properties);
updatedProperties ["message"] = "We're having a heat wave!";
updatedProperties ["temperature"] = 95.0;

var updatedRevision = retrievedDocument.PutProperties (updatedProperties);
System.Diagnostics.Debug.Assert(updatedRevision != null);

Console.WriteLine("Updated document: ");
foreach (var kvp in updatedRevision.Document.Properties){
	Console.WriteLine ("{0} : {1}", kvp.Key, kvp.Value);
}
```

Debug実行をすると、アプリケーション出力にメッセージが出力されます。

```
Updated document: 
message : We're having a heat wave!
created_at : 1/14/2015 9:03:40 AM
_id : 5e2e5087-78e6-44b1-85e7-8698f93614ec
_rev : 2-fb50e7e00bf17c6e02d353ebce929bb8
temperature : 95
```

JSONドキュメントのDELETEをします。

``` csharp:MainActivity.cs
...
// Delete a document
retrievedDocument.Delete ();
Console.WriteLine("Deleted document, deletion status: {0}", retrievedDocument.Deleted);
```

Debug実行をすると、アプリケーション出力にメッセージが出力されます。

```
Deleted document, deletion status: True
```

## MainActivity.cs

最後に完成したコードです。リポジトリは[こちら](https://github.com/masato/HelloCBL)にあります。

``` csharp:MainActivity.cs
using System;

using Android.App;
using Android.Content;
using Android.Runtime;
using Android.Views;
using Android.Widget;
using Android.OS;

using Couchbase.Lite;

using System.Collections.Generic;

namespace HelloCBL
{
	[Activity (Label = "HelloCBL", MainLauncher = true, Icon = "@drawable/icon")]
	public class MainActivity : Activity
	{
		int count = 1;

		protected override void OnCreate (Bundle bundle)
		{
			base.OnCreate (bundle);

			// Set our view from the "main" layout resource
			SetContentView (Resource.Layout.Main);

			// Get our button from the layout resource,
			// and attach an event to it
			Button button = FindViewById<Button> (Resource.Id.myButton);
			
			button.Click += delegate {
				button.Text = string.Format ("{0} clicks!", count++);
			};

			// Create a shared manager
			var manager = Manager.SharedInstance;
			Console.WriteLine ("Manager created");

			// Create database
			var dbName = "hello";
			var database = manager.GetDatabase (dbName);
			Console.WriteLine ("Database created");

			// Create a document
			var properties = new Dictionary <string,object> () {
				{"message","Hello Couchbase Lite"},
				{"created_at",DateTime.UtcNow.ToString("o")},
			};

			var document = database.CreateDocument ();
			var revision = document.PutProperties (properties);
			var docId = document.Id;
			Console.WriteLine ("Document created with ID = {0}", docId);

			var retrievedDocument = database.GetDocument (docId);
			Console.WriteLine ("Retrieved document: ");

			foreach (var kvp in retrievedDocument.Properties) {
				Console.WriteLine ("{0} : {1}", kvp.Key, kvp.Value);
			}

			// Update a document
			var updatedProperties = new Dictionary<string,object> (retrievedDocument.Properties);
			updatedProperties ["message"] = "We're having a heat wave!";
			updatedProperties ["temperature"] = 95.0;

			var updatedRevision = retrievedDocument.PutProperties (updatedProperties);
			System.Diagnostics.Debug.Assert(updatedRevision != null);

			Console.WriteLine("Updated document: ");
			foreach (var kvp in updatedRevision.Document.Properties){
				Console.WriteLine ("{0} : {1}", kvp.Key, kvp.Value);
			}

			// Delete a document
			retrievedDocument.Delete ();
			Console.WriteLine("Deleted document, deletion status: {0}", retrievedDocument.Deleted);

		}
	}
}
```