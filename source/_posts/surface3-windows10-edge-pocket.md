title: "Surface 3に開発環境をつくる - Part1: Microsoft EdgeにPocketのブックマークレット"
date: 2015-07-31 20:36:10
tags:
 - Surface3
 - Windows10
 - MicrosoftEdge
 - Pocket
description: Surface 3をWindows 10にアップグレードして開発環境をつくってみようと思います。私の場合開発はクラウド上のサーバーでEmacsを使うか、Cloud9をブラウザから使っているのでクライアント端末の要件は少ないです。キーボードさえ気に入ればChromebookなどや安価なノートPCで快適な開発環境がつくれる時代になりました。最初にどこでもPocketに積んどくしているので、Windows 10で使えるMicrosoft EdgeでもPocketが使えるようにしてみます。
---


Surface 3をWindows 10にアップグレードして開発環境をつくってみようと思います。私の場合開発はクラウド上のサーバーでEmacsを使うか、Cloud9をブラウザから使っているのでクライアント端末の要件は少ないです。キーボードさえ気に入ればChromebookなど安価なノートPCで快適な開発環境がつくれる時代になりました。最初にどこでも[Pocket](https://getpocket.com)に積んどくしているので、Windows 10で使えるMicrosoft EdgeでもPocketが使えるようにしてみます。


<!-- more -->


## Microsoft Edgeのブックマークレット


他のブラウザの場合、Firefoxではブラウザに統合され、Chromeではエクステンションの[Save to Pocket](https://chrome.google.com/webstore/detail/save-to-pocket/niloccemoadcdkdjlinkgdfekeahmflj?hl=ja)が使えます。しかしMicrosoft Edgeでは現状ブックマークレットを登録する必要があります。困ったことにInternet Explolerにあった「お気に入りの管理」がEdgeには存在しません。[このボタンをブックマークバーにドラッグ。](https://getpocket.com/add/)をした後にURLの変更が画面から行えません。


## Microsoft Edgeのお気に入りの編集

Microsoft Edgeでもお気に入りの管理ができる方法を探していると、ちょうどよいガイドを見つけました。

* [How to manually install bookmarklets in Microsoft Edge ](http://www.itworld.com/article/2954526/personal-technology/how-to-manually-install-bookmarklets-in-microsoft-edge.html)


## Pocketのブックマークレットの作り方

Pocketの[保存方法](https://getpocket.com/add/)を表示します。「またはブックマークレットをインストール ›」をクリックします。「+ Pocket」ボタンを右クリックして「リンクをコピー」してクリップボードに保存しておきます。次に☆マークをクリックしてお気に入りを「お気に入りバー」に作成します。

* 名前: `+ Pocket`
* 作成先: お気に入りバー

お気に入りのURLをブックマークレットのJavaScriptに変更したいのですが、Edgeには編集する画面が見当たりません。[How to manually install bookmarklets in Microsoft Edge ](http://www.itworld.com/article/2954526/personal-technology/how-to-manually-install-bookmarklets-in-microsoft-edge.html)にお気に入りのファイルがどこに保存されているか説明があります。

```
C:\Users\{username}\AppData\Local\Packages\Microsoft.MicrosoftEdge_{random strring}\AC\MicrosoftEdge\User\Default\Favorites\Links
```

エクスプローラーの表示メニューから「隠しファイル」にチェックを入れておきます。{username}と{randam}は環境にあわせて上記のフォルダを開きます。

このフォルダに「+ Pocket」のファイルがあるのでエディタで開きます。URLキーの値をクリップボードにコピーした長いブックマークレットのJavaScriptを貼り付けます。

```txt
URL=javascript:(function(){var e=function(t,n,r,i,s...
```

### お気に入りバー

Edgeの右上にある`three dots menu`から設定を選び、「お気に入りバーを表示する」をオンにします。適当なページを開き、「+ Pocke」のブックマークレットをクリックするとPocketに保存できます。また[マイリスト](https://getpocket.com/a/queue/)もお気に入りバーに追加しておくと便利です。

![pocket.png](/2015/07/31/surface3-windows10-edge-pocket/pocket.png)