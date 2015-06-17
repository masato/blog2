title: "DockerでオープンソースPaaS - Part5: tsuruのDBaaS"
date: 2014-07-06 00:10:43
tags:
 - DockerPaaS
 - DBaaS
 - tsuru
 - MariaDB
 - Flynn
 - Dokku
description: Flynnの場合Flynn PostgreSQLであったり、Dokkuの場合Pluginがあったりと、PaaSにとって何かしらのデータベースサービスが必要です。DokkuのPluginsにはMariaDBやPostgreSQL用のプラグインがあるので、tsuruが理解できたら試してみたいです。オープンソースで作るDBaaSでは、OpenStackのTroveが最近話題です。DBaaSやPaaSをもっとカジュアルに試せるように、そろそろOpenStackもどこかに作らないといけないです。
---

* `Update 2014-08-17`: [Deis in IDCFクラウド - Part4: DBaaS](/2014/08/17/deis-in-idcf-cloud-dbaas/)

Flynnの場合[Flynn PostgreSQL](https://github.com/flynn/flynn-postgres)であったり、Dokkuには[Plugin](https://github.com/progrium/dokku/wiki/Plugins)があったりと、PaaSにとって何かしらのデータベースサービスが必要です。

Dokkuの[Plugins](https://github.com/progrium/dokku/wiki/Plugins)にはMariaDBやPostgreSQL用のプラグインがあるので、tsuruが理解できたら試してみたいです。

オープンソースのDBaaSでは、OpenStackの[Trove](https://github.com/openstack/trove)が最近話題です。
DBaaSやPaaSをもっとカジュアルに試せるように、そろそろOpenStackもどこかに作らないといけないです。


<!-- more -->

### tsuruのサービス

[Services](http://docs.tsuru.io/en/0.5.1/apps/client/services.html)のページに手順が書いてありますが、ドキュメントはページに散らばっているので、わかりづらいのが残念です。

tsuruのバージョンは0.10.2ですが、ドキュメントの最新は0.5.1です。x2?

``` bash
$ tsuru version
tsuru version 0.10.2.
```

[HOWTO Install a MySQL service](http://docs.tsuru.io/en/0.5.1/services/mysql-example.html)にMySQLサービスの作成手順があります。MySQLいってもMariaDBです。


### MariaDBのインストール

MariaDBをインストールします。

``` bash
$ sudo apt-get install software-properties-common
$ sudo apt-key adv --recv-keys --keyserver hkp://keyserver.ubuntu.com:80 0xcbcb082a1bb943db
$ sudo add-apt-repository 'deb http://ftp.yz.yamagata-u.ac.jp/pub/dbms/mariadb/repo/10.1/ubuntu trusty main'
$ sudo apt-get update
$ sudo apt-get install mariadb-server
```

DBの管理ユーザーを作成します。

``` bash
$ mysql -uroot -p
Enter password:
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 43
Server version: 10.1.0-MariaDB-1~trusty-log mariadb.org binary distribution

Copyright (c) 2000, 2014, Oracle, SkySQL Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

> GRANT ALL PRIVILEGES ON *.* TO 'tsuru'@'%' IDENTIFIED BY 'password' with GRANT OPTION;
Query OK, 0 rows affected (0.00 sec)
> FLUSH PRIVILEGES;
Query OK, 0 rows affected (0.00 sec)
```


`bind-address`をコメントアウトして、外部接続を可能にします。

``` bash
$ sudo vi /etc/mysql/my.cnf
#bind-address           = 127.0.0.1
$ sudo /etc/init.d/mysql restart
```

### MySQL service API for tsuru PaaS

MySQLサービスAPIの、[mysqlapi](https://github.com/tsuru/mysqlapi)アプリから利用するデータベースを作成します。

``` bash
$ echo "CREATE DATABASE mysqlapi" | mysql -h 10.1.1.17 -u tsuru -ppassword
```

mysqlapiアプリを`git clone`します。

``` bash
$ git clone https://github.com/globocom/mysqlapi
```

`git push`する前に、tsuruにアプリを作成します。
 
``` bash
$ tsuru app-create mysql-api python
App "mysql-api" is being created!
Use app-info to check the status of the app and its units.
Your repository for "mysql-api" project is "git@10.1.1.17:mysql-api.git"
```

`git push`してmysqlapiアプリをtsuruにデプロイします。

``` bash
$ cd ~/mysqlapi
$ tsuru app-info -a mysql-api|grep Repository
Repository: git@10.1.1.17:mysql-api.git
$ git push git@10.1.1.17:mysql-api.git master
...
remote: Successfully installed Django MySQL-python boto crane-ec2 gunicorn gevent greenlet
remote: Cleaning up...
remote:
remote: ---- Starting 1 unit ----
remote:  ---> Started unit 1/1...
remote:
remote:  ---> App will be restarted, please check its logs for more details...
remote:
remote:
remote: OK
To git@10.1.1.17:mysql-api.git
 * [new branch]      master -> master
```

### 環境変数の設定

手順に環境変数の設定方法があるのですが、まだどのように動作するのかよくわかっていません。

``` bash
$ tsuru env-set -a mysql-api DJANGO_SETTINGS_MODULE=mysqlapi.settings
$ tsuru env-set -a mysql-api MYSQLAPI_DB_NAME=mysqlapi
$ tsuru env-set -a mysql-api MYSQLAPI_DB_USER=tsuru
$ tsuru env-set -a mysql-api MYSQLAPI_DB_PASSWORD=password
$ tsuru env-set -a mysql-api MYSQLAPI_DB_HOST=10.1.1.17
```

この辺りが間違っている気がするのですが、、共有設定をします。

``` bash
$ tsuru env-set --app mysql-api MYSQLAPI_SHARED_SERVER=10.1.1.17
$ tsuru env-set --app mysql-api MYSQLAPI_SHARED_SERVER_PUBLIC_HOST=masato.pw
$ tsuru env-set -a mysql-api MYSQLAPI_SHARED_USER=tsuru
$ tsuru env-set -a mysql-api MYSQLAPI_SHARED_PASSWORD=password
```

### syncdb

一応、Djangoのsyncdbは成功します。tsuruのフォーラムを読んでいると、どうもsyncdbにいっぱい罠がありそうです。

``` bash
$ tsuru run --app mysql-api -- python manage.py syncdb --noinput
Creating tables ...
Creating table auth_permission
Creating table auth_group_permissions
Creating table auth_group
Creating table auth_user_groups
Creating table auth_user_user_permissions
Creating table auth_user
Creating table django_content_type
Creating table django_session
Creating table django_site
Creating table api_instance
Creating table api_provisionedinstance
Installing custom SQL ...
Installing indexes ...
Installed 0 object(s) from 0 fixture(s)
```


### craneでサービスの作成

ここもtsuruのわかりづらいところですが、管理系のコマンドに、別途`crgane`が用意されています。
サービスの追加方法は[api workflow](http://docs.tsuru.io/en/0.5.1/services/api.html)にあります。

craneをインストールします。

``` bash
$ sudo apt-get install crane
```

mysqlqpiのプロジェクトディレクトリに移動して、service.yamlのマニフェストを編集します。

``` bash
$ cd ~/mysqlapi
$ cp service.yaml{,.orig}
$ vi service.yaml
```

ここも怪しいのですが、endpointを指定します。

``` yaml ~/mysqlapi/service.yaml
id: mysqlapi
password: password
endpoint:
  production: mysql-api.masato.pw
  test: localhost:8000
```

サービス作成はcraneコマンドを使います。

``` bash
$ crane create service.yaml
success
```
サービス作成の確認は、`crane list`でも`tsuru service-list`でもリストできます。

``` bash
$ crane list
+----------+-----------+
| Services | Instances |
+----------+-----------+
| mysqlapi |           |
+----------+-----------+
```

### サービスインスタンスの作成

サービスは作成しただけでは使えないので、MySQLサービスインスタンスを作成します。
サービスの作成は`crane create`ですが、インスタンスの作成は`tsuru service-add`です。なんか統一感がないです。

``` bash
$ tsuru service-add mysqlapi blogsql
Service successfully added.
```

サービスインスタンスの確認も、サービスの確認と同じ、`tsuru service-list`コマンドを使います。

``` bash
$ tsuru service-list
+----------+-----------+
| Services | Instances |
+----------+-----------+
| mysqlapi | blogsql   |
+----------+-----------+
```

### まとめ

久しぶりにDjangoを触りましたが、Goの組み合わせは相性がよさそうです。
Goで書けるWebフレームワークが成熟するまで、Go+Djangoはありな気がします。

ResqueもそろそろCeleryにしたいので、またDjangoに戻ろうかと思いました。

サービスインスタンスを作成して、ようやくアプリと連携できるようになります。
次回は、アプリからMySQLサービスを利用する設定をしてみます。




