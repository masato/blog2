title: "DockerのWordPressコンテナをcommitしてもvolumeがimageに含まれなくて困った"
date: 2014-10-01 00:15:03
tags:
 - WordPress
 - Docker
 - OpenResty
 - Flocker
 - xipio
 - nsenter
 - DataVolumeContainer
description: Dockerで構築したWordPressの2-Container-Appコンテナを複製してステージング環境を作ろうとしました。commitしてイメージを作成してからコンテナを起動したのですがvolumeに指定したディレクトリの中身がありません。Data Container volumes can't be saved as images #7583にもissueがありますが現状のデザインのようです。Right, so volumes are not part of the container's fs, they are on the host's fs.In most cases people don't want data in volumes to be in an image, either because it's huge, sensitive information, etc.Flockerみたいな仕組みはあらためて便利に思います。
---

* `Update 2014-10-13`: [MoinMoinをDigitalOceanのCoreOSに移設する - Part2: Dockerイメージの作成](/2014/10/13/docker-moinmoin-data-volume-container-nginx-uwsgi/)
* `Update 2014-10-12`: [MoinMoinをDigitalOceanのCoreOSに移設する - Part1: はじめに](/2014/10/12/docker-moinmoin-design/)


Dockerで構築したWordPressの2-Container-Appを複製してステージング環境を作ろうとしました。
commitしてイメージを作成してからコンテナを起動したのですがvolumeに指定したディレクトリの中身がありません。

[Data Container volumes can't be saved as images #7583](https://github.com/docker/docker/issues/7583)にもissueがありますが現状のデザインのようです。
> Right, so volumes are not part of the container's fs, they are on the host's fs.
In most cases people don't want data in volumes to be in an image, either because it's huge, sensitive information, etc.

[Flocker](/2014/08/21/flocker-with-zfs-on-idcf/)みたいな仕組みはあらためて便利に思います。

<!-- more -->

### volumeバックアップの参考資料

StackOverflowから参考になりそうなポストを探します。

* [Docker volume backup and restore](http://serverfault.com/questions/576490/docker-volume-backup-and-restore)
* [How to port data-only volumes from one host to another?]( http://stackoverflow.com/questions/21597463/how-to-port-data-only-volumes-from-one-host-to-another)

### nsenter

nsenterは[nsenterコマンドをDockerコンテナからホストにマウントして使う](/2014/08/03/docker-volume-nsenter-mount/)でインストールして、nseのラッパーを作成しておきます。

``` bash ~/bin/nse
#!/usr/bin/env bash
[ -n "$1" ] && sudo nsenter --pid --uts --mount --ipc --net --target $(docker inspect --format="&#123;&#123; .State.Pid }}" $1)
```

### コンテナのcommit

複製元になるコンテナをcommitします。ただしvolumeのディレクトリはimageに含まれません。

``` bash
$ docker stop wp
$ docker stop wp-db
$ docker commit wp masato/blog
$ docker commit wp-db masato/blog-db
```

### DBデータコンテナの作成

DBコンテナをinspectしてvolumeを確認します。

``` bash
$ docker inspect --format="&#123;&#123; .Volumes }}" wp-db
map[/etc/mysql:/var/lib/docker/vfs/dir/1d16ed7c668257f65419afec072519ac7d477fa4c0145e134e079ef71fdd7815 /var/lib/mysql:/var/lib/docker/vfs/dir/b4de8453c779df2f69fbe15b77f1adc553e7ef8c087b0b318c2b029f8a668b66]
```

DBコンテナの/var/lib/mysqlと/etc/mysqlをバックアップをします。

``` bash
$ docker run --rm --volumes-from wp-db -v $(pwd):/backup busybox tar cvf /backup/var_lib_mysql.tar /var/lib/mysql
$ docker run --rm --volumes-from wp-db -v $(pwd):/backup busybox tar cvf /backup/etc_mysql.tar /etc/mysql
```

nsenterで接続したいのでbusyboxでなくubuntuでデータコンテナを作成します。
作成したデータコンテナにDockerホストのローカルにバックアップしたtarをuntarします。

``` bash
$ docker run -i -t -d -v /var/lib/mysql -v /etc/mysql --name db-stg-vol ubuntu /bin/bash
$ docker run --rm --volumes-from db-stg-vol -v $(pwd):/backup busybox tar xvf /backup/var_lib_mysql.tar
...
var/lib/mysql/performance_schema/events_waits_history_long.frm
var/lib/mysql/performance_schema/file_summary_by_instance.frm
var/lib/mysql/ibdata1
$ docker run --rm --volumes-from db-stg-vol -v $(pwd):/backup busybox tar xvf /backup/etc_mysql.tar
...
etc/mysql/conf.d/my.cnf
etc/mysql/conf.d/mysqld_safe_syslog.cnf
etc/mysql/debian-start
```

db-stg-volコンテナをマウントしてdb-stgコンテナを起動します。

``` bash
$ docker run --name db-stg -d --volumes-from db-stg-vol masato/blog-db
```

nsenterでdb-stgに接続して、DBコンテナが正常に複製できたかmysqlの確認をします

``` bash
$ nse db-stg
$ mysql -uadmin -ppassword wordpress
mysql>
```


### WordPressデータコンテナの作成

wpコンテナのボリューム(/app/wp-content)をバックアップして、wp-stg-volにリストアします。

``` bash
$ docker run --rm --volumes-from wp -v $(pwd):/backup busybox tar cvf /backup/app_wp_content.tar /app/wp-content
$ docker run -i -t -d -v /app/wp-content --name wp-stg-vol ubuntu /bin/bash
$ docker run --rm --volumes-from wp-stg-vol -v $(pwd):/backup busybox tar xvf /backup/app_wp_content.tar
...
app/wp-content/themes/responsive.orig/search.php
app/wp-content/themes/responsive.orig/sidebar-right.php
app/wp-content/themes/responsive.orig/404.php
```

db-stgコンテナのIPアドレスの確認をします。

``` bash
$ docker inspect --format="&#123;&#123; .NetworkSettings.IPAddress }}" db-stg
172.17.0.128
```

wp-stgコンテナを起動します。

```
$ docker run -d --link db-stg:db -e DB_HOST="172.17.0.128" -e DB_PORT="3306" -e DB_PASS="password" --name wp-stg --volumes-from wp-stg-vol -p 8091:80 masato/blog
```

このイメージは一度DBを作成した状態から作成しています。すでに/.mysql_db_createdが存在しているので/run.shを実行後にexitしてしまいます。そのためDB_HOSTとDB_HOSTの環境変数がlinkingから取得して設定できないため、上記のrunでは明示的に環境変数をしていしています。

``` bash run-wordpress.sh
#!/bin/bash
if [ -f /.mysql_db_created ]; then
        exec /run.sh
        exit 1
fi

DB_HOST=${DB_PORT_3306_TCP_ADDR:-${DB_HOST}}
DB_HOST=${DB_1_PORT_3306_TCP_ADDR:-${DB_HOST}}
DB_PORT=${DB_PORT_3306_TCP_PORT:-${DB_PORT}}
DB_PORT=${DB_1_PORT_3306_TCP_PORT:-${DB_PORT}}
```

使い捨てのコンテナで、どのような環境変数が設定される確認します

``` bash
$ docker run --rm -i -t  --link db-stg:db -e DB_HOST="172.17.0.128" -e DB_PORT="3306" -e DB_PASS="password"  --volumes-from wp-stg-vol  idcf/engineer-blog env
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
HOSTNAME=f7f9091680dd
TERM=xterm
DB_PORT=3306
DB_PORT_3306_TCP=tcp://172.17.0.128:3306
DB_PORT_3306_TCP_ADDR=172.17.0.128
DB_PORT_3306_TCP_PORT=3306
DB_PORT_3306_TCP_PROTO=tcp
DB_NAME=wordpress
DB_ENV_MYSQL_PASS=password
DB_ENV_DEBIAN_FRONTEND=noninteractive
DB_ENV_MYSQL_USER=admin
DB_HOST=172.17.0.128
DB_PASS=password
HOME=/
DEBIAN_FRONTEND=noninteractive
DB_USER=admin
```

wp-stgコンテナにnsenterで接続してリバースプロキシ用に、これから作成するxipのドメインを指定します。

``` bash
$ nse wp-stg
$ vi /app/wp-config.php
define('WP_SITEURL', 'http://p8091.210.140.17.194.xip.io/blog');
define('WP_HOME', 'http://p8091.210.140.17.194.xip.io/blog/');
define('WP_CONTENT_URL', 'http://p8091.210.140.17.194.xip.io/blog/wp-content');
```

OpenRestyでリバースプロキシのHTTPルーティングを指定します。

``` bash
$ docker restart wp-stg
$ docker inspect --format="&#123;&#123; .NetworkSettings.IPAddress }}" wp-stg
172.17.0.141
$ docker run -it --rm --link openresty:redis dockerfile/redis bash -c 'redis-cli -h $REDIS_PORT_6379_TCP_ADDR set p8091.210.140.17.194.xip.io 172.17.0.141:80'
```

 * xipのドメインで複製したコンテナの起動を確認します。

 http://p8091.210.140.17.194.xip.io/blog

