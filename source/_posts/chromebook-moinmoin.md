title: 'ChromebookのUbuntuにMoinMoinをインストール'
date: 2014-05-05 21:32:44
tags:
 - Chromebook
 - MoinMoin
description: 作業記録はMoinMoinのWikiに書いています。Chromebookでオフラインな時もWikiを参照したいです。定期的にAmazonS3にバックアップしているpagesを、新しいサイトにリストアします。
---

* `Update 2014-10-12`: [MoinMoinをDigitalOceanのCoreOSに移設する - Part1: はじめに](/2014/10/12/docker-moinmoin-design/)

作業記録は[MoinMoin](http://moinmo.in/)のWikiに書いています。Chromebookでオフラインな時もWikiを参照したいです。
定期的にAmazonS3にバックアップしているpagesを、新しいサイトにリストアします。

<!-- more -->

## MoinMoinのインストール

MoinMoinを起動するため、Apacheとmod_wsgiをインストールします。

``` bash
$ sudo apt-get install apache2 libapache2-mod-wsgi
```
MoinMoinは1.9.x系をインストールします。

``` bash
$ cd ~/Downloads/
$ wget http://static.moinmo.in/files/moin-1.9.7.tar.gz
$ tar xvzf moin-1.9.7.tar.gz
$ cd moin-1.9.7
$ sudo python setup.py install --force --prefix=/usr --record=install.log
```

以下のディレクトリにインストールされました。

```
/usr/lib/python2.7/site-packages/MoinMoin
```

### WSGIのテスト
WSGIのテストサーバーを起動します。テストーサーバーはCtrl+Cで停止します。

``` bash
$ cd /usr/share/moin/server
$ sudo python test.wsgi
```

Webブラウザで確認
http://localhost:8000/


### MoinMoinの設定

設定ファイルをサイトのディレクトリにコピーします。

``` bash
$ cd /usr/share/moin
$ sudo cp server/moin.wsgi .
$ sudo cp config/wikiconfig.py .
```

wsgi.confの設定

``` python /etc/apache2/mods-enabled/wsgi.conf
#
#  MoinMoin WSGI configuration
#
# you will invoke your moin wiki at the root url, like http://servername/FrontPage:
WSGIScriptAlias /   /usr/share/moin/moin.wsgi

# create some wsgi daemons - use these parameters for a simple setup
WSGIDaemonProcess moin user=www-data group=www-data processes=5 threads=10 maximum-requests=1000 umask=0007

# use the daemons we defined above to process requests!
WSGIProcessGroup moin
```

WSGIファイルにsys.pathを追加します。
``` python /usr/share/moin/moin.wsgi
sys.path.insert(0, '/usr/share/moin')
sys.path.insert(0, '/usr/lib/python2.7/site-packages')
```

パーミッションを設定します。

``` bash
$ cd /usr/share
$ sudo chown -R www-data:www-data moin
$ sudo chmod -R ug+rwX moin
$ sudo chmod -R o-rwx moin
```

wikiconfig.pyに基本的な設定を書きます。

``` python /usr/share/moin/wikiconfig.py
sitename = u'Wiki'
page_front_page = u"FrontPage"
superuser = [u"MasatoShimizu", ]
acl_rights_default = u"MasatoShimizu:read,write,delete,revert,admin"
theme_default = 'europython'
tz_offset = 9.0
```

### テーマ

MoinMoinの[ThemeMarket](http://moinmo.in/ThemeMarket)から気に入ったテーマをダウンロードします。
1.9+に対応している、[Europython](http://moinmo.in/ThemeMarket/Europython)を使います。

カスタマイズ用にhtdocsをコピーします。

``` bash
$ sudo cp -R /usr/lib/python2.7/site-packages/MoinMoin/web/static/htdocs /usr/share/moin
```

コピーしたhtdocsを使うように設定します。

``` python /usr/share/moin/moin.wsgi
application = make_application(shared='/usr/share/moin/htdocs')
```

Europythonダウンロードして、サイトにコピーします。

``` bash
$ cd ~/Downloads
$ wget https://bitbucket.org/thesheep/moin-europython/get/3995afe116b0.zip
$ unzip 3995afe116b0.zip
$ cd ~/Downloads/thesheep-moin-europython-3995afe116b0
$ sudo cp europython.py /usr/share/moin/data/plugin/theme
$ sudo mkdir -p /usr/share/moin/htdocs/europython
$ sudo cp -R css /usr/share/moin/htdocs/europython
$ sudo cp -R img /usr/share/moin/htdocs/europython
```

パーミッションを設定します。

``` bash
$ cd /usr/share
$ sudo chown -R www-data.www-data /usr/share/moin/data/plugin/theme/europython.py
$ sudo chown -R www-data.www-data /usr/share/moin/htdocs
$ sudo chmod -R ug+rwX /usr/share/moin/htdocs
$ sudo chmod -R o-rwx /usr/share/moin/htdocs
```

### font-familyのカスタマイズ

sidebarのsans-serifをコメントアウトします。

``` css /usr/share/moin/htdocs/europython/css/screen.css
html {
    font-family:  "lucida grande",tahoma,verdana,arial,'ヒラギノ角ゴ Pro W3', 'Hiragino Kaku Gothic Pro', 'Osaka', 'Meiryo UI','メイリオ', 'ＭＳ Ｐゴシック', sans-serif;
}
div#sidebar ul#navibar li a {
  /*font-family: Verdana,Geneva,"Bitstream Vera Sans",Helvetica,sans-serif;*/
}
```

ヘッダーのserifをコメントアウトします。

``` css /usr/share/moin/htdocs/europython/css/common.css
h1, h2, h3, h4, h5 {
    /*font-family: serif;*/
    font-weight: normal;
    letter-spacing: 0.05em;
}
```

### デフォルトのpagesで起動の確認

Apacheを再起動します。

``` bash
$ sudo service apache2 restart
```

ブラウザで確認します。
http://localhost

### プリファレンス

LoginをクリックしWikiAdminユーザーを作成します。
ここでは、superuserの、MasatoSHimizuを作成してログインします。

### pagesのリストア

AmazonS3に定期的にバックアップしているpagesをs3cmdでダウンロードします。

``` bash
$ sudo apt-get install s3cmd
$ s3cmd --configure
```

バックアップの一覧を表示します。

``` bash
$ s3cmd  ls s3://ma5ato/moin/ | sort -r
2014-05-11 11:52  49722874   s3://ma5ato/moin/140511205002_pages.tar.gz
2014-05-11 07:52  49721997   s3://ma5ato/moin/140511165002_pages.tar.gz
2014-05-11 03:52  49719565   s3://ma5ato/moin/140511125002_pages.tar.gz
...
```

最新のpagesをダウンロードします。

``` bash
$ cd ~/Downloads
$ s3cmd get  s3://ma5ato/moin/140511205002_pages.tar.gz
```

デフォルトのpagesのバックアップをとってから、pagesをサイトに展開します。

``` bash
$ sudo mv /usr/share/moin/data/pages /usr/share/moin/data/pages.orig
$ sudo tar zxvf 140511205002_pages.tar.gz -C /usr/share/moin/data/
$ sudo chown -R www-data.www-data /usr/share/moin/data/pages
```

ブラウザで確認します。
http://localhost

### まとめ

オンラインのときに最新のpagesをダウンロードする必要がありますが、オフラインのChromebookでもWikiを参照できるようになりました。
オフラインのときでも、Wikiを編集したいので、次はpagesを同期する仕組みを考えてみます。
