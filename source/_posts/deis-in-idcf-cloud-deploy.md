title: "Deis in IDCFクラウド - Part2: SinatraをDeisにデプロイ"
date: 2014-08-15 00:55:55
tags:
 - Deis
 - CoreOS
 - IDCFクラウド
 - GCE
 - xipio
 - Sinatra
description: 構築したDeisの設定と、簡単なアプリのデプロイまで行ってみます。Deis in Google Compute Engineで使っている、Google Cloud DNSの代わりにxip.ioを、Load Balancingの代わりに、IDCFクラウドに標準機能であるロードバランサーを利用します。
---

[構築したDeis](/2014/08/14/deis-in-idcf-cloud-install/)の設定と、簡単なアプリのデプロイまで行ってみます。

[Deis in Google Compute Engine](https://gist.github.com/andyshinn/a78b617b2b16a2782655)で使っている、[Google Cloud DNS](https://developers.google.com/cloud-dns/?hl=ja)の代わりに[xip.io](http://xip.io/)を、[Load Balancing](https://developers.google.com/compute/docs/load-balancing/)の代わりに、IDCFクラウドに標準機能である[ロードバランサー](http://www.idcf.jp/cloud/self/manual/03.html)を利用します。

<!-- more -->

### Deisクライアントのインストール

作業マシンにDeisクライアントをインストールします。バージョンは0.10.0です。

```
$ sudo pip install --upgrade deis
$ deis --version
Deis CLI 0.10.0
$ which deis
/usr/local/bin/deis
```

### ネットワーク設定

2222ポートと80ポートのロードバランサーを設定します。
roundrobinの分散対象にはCoreOSクラスタの3台を指定します。

WildcardDNSはxip.ioを使うため、Deisの命名規則に従い以下のドメイン名にしました。

```
deis.210.140.16.229.xip.io
```

### Deisコントローラーへユーザー登録

Deisコントローラーにユーザーを登録します。

```
$ deis register http://deis.210.140.16.229.xip.io
username: masato
password:
password (confirm):
email: ma6ato@gmail.com
Registered masato
Logged in as masato
```

`deis keys:add`を入力すると、`~/.ssh`に存在する鍵をリストしてくれるので、
Deisで利用したい鍵の番号を選択します。

``` 
$ deis keys:add 
```

以降の管理作業をするため、Deisコントローラーにログインします。

```
$ deis login http://deis.210.140.16.229.xip.io
username: masato
password:
Logged in as masato
```

### Deisクラスタの作成

アプリケーションをデプロイするために、Deisのクラスタを作成します。
`--hosts`がわかりづらいところですが、CoreOSの内部IPアドレスを指定します。

```
$ deis clusters:create dev deis.210.140.16.229.xip.io \
    --hosts=10.1.2.34,10.1.3.33,10.1.0.249 \
    --auth=~/.ssh/deis
Creating cluster... done, created dev
```

### Sinatraのデプロイ

Deisが提供しているSinatraの[サンプルアプリ](https://github.com/deis/example-ruby-sinatra.git)をcloneします。

```
$ git clone https://github.com/deis/example-ruby-sinatra.git
$ cd example-ruby-sinatra
```

アプリケーションを作成します。Gitのリモートリポジトリが追加されました。

```
$ deis create
Creating application... done, created outlaw-rucksack
Warning: Permanently added '[deis.210.140.16.229.xip.io]:2222,[210.140.16.229]:2222' (ECDSA) to the list of known hosts.
Git remote deis added
```

`git push`をしてデプロイします。ログをみるとHerokuのようにslugを作成してからデプロイしています。

```
$ git push deis master
Counting objects: 102, done.
Delta compression using up to 4 threads.
Compressing objects: 100% (53/53), done.
Writing objects: 100% (102/102), 22.19 KiB | 0 bytes/s, done.
Total 102 (delta 43), reused 102 (delta 43)
-----> Ruby app detected
-----> Compiling Ruby/Rack
-----> Using Ruby version: ruby-1.9.3
-----> Installing dependencies using 1.6.3
       Running: bundle install --without development:test --path vendor/bundle --binstubs vendor/bundle/bin -j4 --deployment
       Don't run Bundler as root. Bundler can ask for sudo if it is needed, and
       installing your bundle as root will break this application for all non-root
       users on this machine.
       Fetching gem metadata from http://rubygems.org/..........
       Using bundler 1.6.3
       Installing tilt 1.3.6
       Installing rack 1.5.2
       Installing rack-protection 1.5.0
       Installing sinatra 1.4.2
       Your bundle is complete!
       Gems in the groups development and test were not installed.
       It was installed into ./vendor/bundle
       Bundle completed (13.48s)
       Cleaning up the bundler cache.
-----> Discovering process types
       Procfile declares types -> web
       Default process types for Ruby -> rake, console, web
-----> Compiled slug size is 12M
remote: -----> Building Docker image
remote: Sending build context to Docker daemon 11.77 MB
remote: Sending build context to Docker daemon
remote: Step 0 : FROM deis/slugrunner
remote:  ---> f607bc8783a5
remote: Step 1 : RUN mkdir -p /app
remote:  ---> Running in 0b97b30b3657
remote:  ---> 78bff4f79476
remote: Removing intermediate container 0b97b30b3657
remote: Step 2 : ADD slug.tgz /app
remote:  ---> 022a38002b84
remote: Removing intermediate container d1b6139bd64c
remote: Step 3 : ENTRYPOINT ["/runner/init"]
remote:  ---> Running in 19ae5bcceb39
remote:  ---> abdcc4adba9a
remote: Removing intermediate container 19ae5bcceb39
remote: Successfully built abdcc4adba9a
remote: -----> Pushing image to private registry
remote:
remote:        Launching... done, v2
remote:
remote: -----> outlaw-rucksack deployed to Deis
remote:        http://outlaw-rucksack.deis.210.140.16.229.xip.io
remote:
remote:        To learn more, use `deis help` or visit http://deis.io
remote:
To ssh://git@deis.210.140.16.229.xip.io:2222/outlaw-rucksack.git
 * [new branch]      master -> master
```

### 確認

`git push deis master`ログの最後の方にアプリのURLが表示されるので、curlで確認してみます。

```
$ curl -s http://outlaw-rucksack.deis.210.140.16.229.xip.io
Powered by Deis!
Running on container ID 3110024e035b
```

ブラウザからも確認してみます。

{% img center /2014/08/15/deis-in-idcf-cloud-deploy/deis.png %}



