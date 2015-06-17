title: 'Nitrous.IOはじめました'
date: 2014-04-30 00:57:13
tags:
 - Nodejs
 - Chromebook
 - NitrousIO
 - IDE
description: ChromebookとNitrous.IOを使っています。Yehuda Katzがおすすめなのもポイントです。2014年1月くらいの時は、コピペしないと日本語入力できないので要望をあげていたら、いつの間にか日本語も直接入力できるようになりました。
---
[The Homeless Man](http://www.businessinsider.com/homeless-coder-2013-9)みたいな開発環境を作りたくて、Chromebookと[Nitrous.IO](https://www.nitrous.io/)を使っています。`Yehuda Katz`が[おすすめ](http://blog.nitrous.io/2013/08/05/nitrous-stories-i-yehuda-katz-tilde-nitrousio.html)なのもポイントです。
2014年1月くらいの時は、コピペしないと日本語入力できないので要望をあげていたら、いつの間にか日本語も直接入力できるようになりました。
<!-- more -->

### Box

[Box](http://help.nitrous.io/box-new/)と呼ばれるコンテナを作成して開発を行います。いくつかテンプレートが用意されているので、GoやNode.jsの開発がすぐにできるようになっています。
テンプレートを使わなくても、あとから[Autoparts](http://help.nitrous.io/autoparts/)というパッケージマネージャーを使ってDBやツールをインストールできます。[パッケージ](https://github.com/nitrous-io/autoparts/tree/master/lib/autoparts/packages)はたくさん用意されています。

### Goで開発ができるCloudIDE

Chromebookで使えるCloudIDEやWebIDEは最近増えています。Goが対応していないと使いたくないので、こんな候補になりました。
SecureShellで日本語が入力できるようになれば、VPSに接続してEmacsを使えばいいだけかも知れませんが、PaaSですぐ開発できるCloudIDEは捨てがたいです。

* [Nitrous.IO](https://www.nitrous.io/)
* [Codebox](https://www.codebox.io/)
* [Codio](https://codio.com/)
* [Koding](https://koding.com/)
* [Godev](https://github.com/sirnewton01/godev)

### Nitrous.IOのよいところ

Nitrous.IOを使っているのは、次の理由からです。

* DBを使わないBoxなら、無料プランで使える。
* Boxの起動が速い。(有償プランならBoxは常時起動しておける)
* [プレビュー](http://help.nitrous.io/preview/)機能がある。
* East Asiaリージョンが選べるので、レイテンシが小さい。

特にプレビューは、自動で振られるドメイン名を使い、Chromeの別タブから確認できるので、開発時にはとても便利です。

### まとめ

オフラインでは使えませんが、テザリングや公衆無線LANを使うと外出時でもオンラインでいられることが多くなったので、CloudIDEも現実味がでてきたと思います。
特に3万円以下で買えるChromebookとの組み合わせは、発展途上国や教育機関でプロブラミングを学習するのに最適です。
