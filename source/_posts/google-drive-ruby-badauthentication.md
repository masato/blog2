title: google-drive-rubyの0.3.x系でBadAuthenticationの対応
date: 2015-02-02 12:20:20
tags:
 - Ruby
 - GoogleDrive
 - GoogleSpreadSheet
 - google-drive-ruby
description: Googleスプレッドシートをテンプレートに使いPDF帳票作成のWebアプリを動かしています。本番環境でしばらく安定して動いていたのですが、先日PDFが作成されなくなったと連絡がありました。Googleアカウントの2段階認証プロセスを有効にしたあとに、認証エラーになっているようです。
---

Googleスプレッドシートをテンプレートに使いPDF帳票作成のWebアプリを動かしています。本番環境でしばらく安定して動いていたのですが、先日PDFが作成されなくなったと連絡がありました。Googleアカウントの2段階認証プロセスを有効にしたあとに、認証エラーになっているようです。

<!-- more -->

## google-drive-rubyのバージョン

[README.rdoc](https://github.com/gimite/google-drive-ruby/blob/master/README.rdoc)に注意書きがあります。1.0.0の[バージョン](https://rubygems.org/gems/google_drive)になり0.3.xと上位互換が完全でありません。特にloginメソッドがなくなっているので注意が必要です。

> Ver. 1.0.0 is not 100% backward compatible with 0.3.x. Some methods have been removed. Especially, GoogleDrive.login has been removed, and you must use GoogleDrive.login_with_oauth instead, as in the example code below.

## GoogleDrive.loginで認証に失敗

今回の環境のバージョンです。

``` bash
$ bundle exec ruby -v
ruby 1.9.3p327 (2012-11-10 revision 37606) [x86_64-linux]
$ bundle exec gem list google_drive

*** LOCAL GEMS ***

google_drive (0.3.6)
```

コンソールで再現してみると、GoogleDrive.loginで認証に失敗しています。

``` ruby
$ bundle exec rails console 
irb(main):001:0> session = GoogleDrive.login 'Googleアカウント','Googleアカウントのパスワード'
GoogleDrive::AuthenticationError: Authentication failed for xxx Response code 403
for post https://www.google.com/accounts/ClientLogin: Error=BadAuthentication
```

## アプリ パスワードでログイン

Googleアカウントの[ヘルプページ](https://support.google.com/accounts/answer/185833?hl=ja)によると、2段階認証プロセスを有効にした後はアプリ パスワードに変更する必要があります。

> アプリ パスワードは、Google アカウントへのアクセス権をアプリや端末に付与する 16 桁のパスコードです。2 段階認証プロセスを使用している場合に、Google アカウントにアクセスして「パスワードが正しくありません」というエラーが表示されるときは、アプリ パスワードで問題を解決できることがあります。

[アプリ パスワード](https://security.google.com/settings/security/apppasswords)の画面から新しく16 桁のパスコードを生成して、Googleアカウントのパスワードの代わりに使います。

## PDF出力テスト

新しく生成した16 桁のパスコードを使い、ローカルにPDFを出力してさっとテストをしてみます。無事にPDF出力ができました。今回はプログラムの修正をせずに済んだのですが、早めにGemのバージョンアップをして、OAuth認証に移行しないといけません。

``` ruby
$ bundle exec rails console 
irb(main):001:0> session = GoogleDrive.login 'Googleアカウント','16 桁のパスコード'
irb(main):002:0> template_key = 'ドキュメントキー'
irb(main):003:0> new_title = "test_#{Time.now.strftime('%Y%m%d%H%M%S')}"
irb(main):004:0> template = session.spreadsheet_by_key template_key
irb(main):005:0> sheet = template.duplicate(new_title=new_title)
irb(main):006:0> key = sheet.key
irb(main):007:0> format = 'pdf'
irb(main):008:0> gid_param = '&gridlines=false&printtitle=false&size=A4&fzr=true&portrait=true&fitw=true'
irb(main):009:0> url = ["https://spreadsheets.google.com/feeds/download/spreadsheets/Export",\
irb(main):010:1* "?key=#{key}&exportFormat=#{format}#{gid_param}"].join
irb(main):011:0> raw_data = session.request(:get, url, :response_type => :raw)
irb(main):012:0> File.binwrite("/tmp/#{new_title}.pdf", raw_data)
irb(main):013:0> session.file_by_title(new_title).delete
```






