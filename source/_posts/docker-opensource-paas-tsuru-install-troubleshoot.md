title: "DockerでオープンソースPaaS - Part4: tsuruインストールトラブルシュート"
date: 2014-07-05 21:05:58
tags:
 - Docker
 - DockerPaaS
 - tsuru
description: tsusuをTsuru Nowでインストールした後、オプションを変更して`/etc/tsuru/tsuru.conf`の設定を変更したかったり、なにか問題があってインストールが中断することがあります。ワンライナーのインストーラーを使うと、手順が見えにくいので再インストールが難しくなり、結局スクリプトを細かくデバッグすることになります。便利なのですが、インストール前の状態に依存するところもあるので、自分でワンライナーインストーラー作成の難しさを感じます。
---

[tsusu](https://github.com/tsuru/tsuru)を[Tsuru Now](https://github.com/tsuru/now)でインストールした後、
オプションを変更して`/etc/tsuru/tsuru.conf`の設定を変更したかったり、なにか問題があってインストールが中断することがあります。

ワンライナーのインストーラーを使うと、手順が見えにくいので再インストールが難しくなり、
結局スクリプトを細かくデバッグすることになります。

便利なのですが、インストール前の状態に依存するところもあるので、自分でワンライナーインストーラー作成の難しさを感じます。


<!-- more -->

### アプリのアンインストール

まず、trusu-dashboardなどアプリを削除します。

``` bash
$ tsuru app-list
+-----------------+-------------------------+---------------------------+--------+
| Application     | Units State Summary     | Address                   | Ready? |
+-----------------+-------------------------+---------------------------+--------+
| blog            | 1 of 1 units in-service | blog.masato.pw            | Yes    |
| mysql-api       | 1 of 1 units in-service | mysql-api.masato.pw       | Yes    |
| tsuru-dashboard | 1 of 1 units in-service | tsuru-dashboard.masato.pw | Yes    |
+-----------------+-------------------------+---------------------------+--------+
$ tsuru app-remove -a blog
$ tsuru app-remove -a mysql-api
$ tsuru app-remove -a tsuru-dashboard
```

### サービスの削除

`tsuru service-list`コマンドで作成しているサービスを削除します。

``` bash
$ tsuru service-list
+----------+-----------+
| Services | Instances |
+----------+-----------+
| mysqlapi | blogsql   |
+----------+-----------+
$ tsuru service-remove blogsql
Are you sure you want to remove service "blogsql"? (y/n) y
Service "blogsql" successfully removed!
```

正常にアプリが削除されると、Dockerコンテナも削除されています。

```
$ docker  ps -a
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
```

イメージのビルドに失敗した場合は削除します。

``` bash
$ docker rmi {イメージID}
```

### MariaDBの削除

MariaDBを停止して削除します。

``` bash
$ sudo /etc/init.d/mysql stop
$ sudo apt-get remove --purge mariadb-server
$ sudo apt-get autoremove
$ sudo rm -rf /var/lib/mysql
```


### パッケージの削除

apt-getでインストールしたパッケージを削除します。

``` bash
$ sudo apt-get -y remove --purge gandalf-server mongodb-org-server redis-server mongodb-org-mongos mongodb-org-shell mongodb-org-tools redis-tools lxc-docker-1.1.0  mariadb-server
$ sudo apt-get -y autoremove
```

### Hipacheの削除

Hipacheはnpmでグローバルにインストールされています。

``` bash
$ sudo NODE_PATH=/usr/lib/node_modules npm uninstall hipache -g
unbuild hipache@0.3.1
```

### 各種インストールディレクトリを削除します。

tsuru, MongoDB, Redis, Docker, Gandalf, Go, tsuru-dashboardのインストールディレクトリを削除します。

``` bash
$ sudo rm -fr /var/lib/tsuru/ /var/lib/redis/ /var/lib/mongodb/ /var/lib/docker/  /var/lib/gandalf/ ~/go /tmp/tsuru-dashboard/
```

### Tsuruの設定ファイル

/etc配下の設定ファイルを削除します。

``` bash
$ sudo rm /etc/tsuru/tsuru.conf
```


### gitユーザーの.bash_profile削除

再インストールしてもなぜかアプリのデプロイの`git push`失敗して、かなり嵌まりました。
デバッグしていると、gitユーザーの`.bash_profile`に`TSURU_HOST`と`TSURU_TOKEN`が再インストールするたびに追加されてしまい、不正な状態になっていました。

gitユーザーの`.bash_profile`を削除します。

``` bash
$ sudo rm /home/git/.bash_profile
```

### Hipacheの事前インストール

ここも嵌まりました。

なぜかHipacheは次回の`Tsuru Now`のインストールスクリプトでは再インストールしてくれないので、手動でインストールします。

``` bash
$ sudo NODE_PATH=/usr/lib/node_modules npm install hipache -g
```

### まとめ

構造を理解すればどうということはないトラブルシュートですが、始めはすごく嵌まる可能性が高いです。
インストールのbashはわかりやすいので、丁寧にデバッグしていけば解決できますが、
ドキュメントはもうちょっとわかりやすく書いて欲しいです。

あと、アンインストールもワンライナーでできるといいのにと思います。



