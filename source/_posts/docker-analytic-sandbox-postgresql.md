title: "Dockerでデータ分析環境 - Part3: PostgreSQLとボリューム"
date: 2014-07-12 06:45:10
tags:
 - Docker
 - AnalyticSandbox
 - PostgreSQL
 - DBaaS
 - Dokku
 - runit
 - DataVolumeContainer
description: データベースもインフラから分離して自由にアタッチできるDBaaS環境を探しています。さらにデータ自体もデータベースコンテナから分離できたり、Dockerの世界は奥が深いです。Data-Only-Containerの使い方は、How to port data-only volumes from one host to another?や、オフィシャルのManaging Data in Containersを読みながら勉強していこうと思います。tsuruのMySQLはまだうまく使えず、今回の要件はデータ分析の学習環境なので簡単なところから始めます。DBを起動するコンテナとDockerホストへのボリュームのマップを行います。
---

* `Update 2014-07-13`: [Dockerでデータ分析環境 - Part4: Tomcat7とPostgreSQL Studio](/2014/07/13/docker-analytic-sandbox-postgresql-studio/)

データベースもインフラから分離して自由にアタッチできるDBaaS環境を探しています。
さらにデータ自体もデータベースコンテナから分離できたり、Dockerの世界は奥が深いです。

Data-Only-Containerの使い方は、[How to port data-only volumes from one host to another?](http://stackoverflow.com/questions/21597463/how-to-port-data-only-volumes-from-one-host-to-another)や、オフィシャルの[Managing Data in Containers](http://docs.docker.com/userguide/dockervolumes/)を読みながら勉強していこうと思います。

[tsuru](/2014/07/08/docker-opensource-paas-tsuru-go/)のMySQLはまだうまく使えず、今回の要件はデータ分析の学習環境なので簡単なところから始めます。
DBを起動するコンテナとDockerホストへのボリュームのマップを行います。

<!-- more -->

### paintedfox/postgresql

Dockerファイルは[paintedfox/postgresql](https://registry.hub.docker.com/u/paintedfox/postgresql/dockerfile)を使います。GitHubは[Painted-Fox/docker-postgresql](https://github.com/Painted-Fox/docker-postgresql)です。

### Dockerfile

Dockerfileのbaseimageは、いつもお世話になっている[phusion/baseimage](https://github.com/phusion/baseimage-docker)なのでわかりやすいです。

ロケールが`en_US.UTF-8`だったり、SSH接続を無効にしているので、確認ができたら修正して使おうと思います。

### runit

ruintを使っている人はあまりいないようで、PostgreSQLのrunitの書き方を探していて、
このDockerfileをみつけたのがきっかけです。データベースの場合、起動スクリプトの書き方が複雑になります。

### 使い方

READMEに使い方が書いてあるので参考にしながらオプションを確認します。

Dockerホスト上にコンテナにマップするディレクトリを作ります。

``` bash
$ sudo mkdir -p /opt/docker_vols/postgresql
$ cd !$
```

Dockerコンテナを起動します。中身を確認したいので、仮想端末に接続してbashを起動します。

```
$ docker run --rm -i -t --name="postgresql" \
             -p 5432:5432 \
             -v /opt/docker_vols/postgresql:/data \
             -e USER="masato" \
             -e DB="spike_db" \
             -e PASS="password" \
             paintedfox/postgresql /sbin/my_init bash
...
*** Running /etc/rc.local...
*** Booting runit daemon...
*** Runit started as PID 12
*** Running bash...
POSTGRES_USER=masato
POSTGRES_PASS=password
POSTGRES_DATA_DIR=/data
POSTGRES_DB=spike_db
Starting PostgreSQL...
root@75ac0e3f6e44:/# Creating the superuser: masato
Creating database: spike_db

root@75ac0e3f6e44:/#
```

psqlコマンドでデータベースへの接続を確認します。

``` bash
# psql -U masato -d spike_db -h localhost
Password for user masato:
psql (9.3.4)
SSL connection (cipher: DHE-RSA-AES256-SHA, bits: 256)
Type "help" for help.

spike_db=#
```

postgresql.confをみると、`data_directory`は`/data`を指定しています。
コンテナの起動時の`-v`オプションで、Dockerホストのディレクトリへマップしています。

``` bash /etc/postgresql/9.3/main/postgresql.conf
data_directory = '/data'
```

### Dockerホストのマップされたディレクトリ

Dockerコンテナからマップしたディレクトリです。
PostgreSQLのコンテナを破棄すると`postmaster.pid`も消去されます。


``` bash
$ sudo ls -al /opt/docker_vols/postgresql/
合計 72
drwx------ 15 usbmux root   4096  7月 12 09:39 .
drwxr-xr-x  3 root   root   4096  7月 12 08:51 ..
-rwx------  1 usbmux root      4  7月 12 08:51 PG_VERSION
drwx------  6 usbmux root   4096  7月 12 08:51 base
drwx------  2 usbmux root   4096  7月 12 09:39 global
drwx------  2 usbmux root   4096  7月 12 08:51 pg_clog
drwx------  4 usbmux root   4096  7月 12 08:51 pg_multixact
drwx------  2 usbmux root   4096  7月 12 09:39 pg_notify
drwx------  2 usbmux root   4096  7月 12 08:51 pg_serial
drwx------  2 usbmux root   4096  7月 12 08:51 pg_snapshots
drwx------  2 usbmux root   4096  7月 12 09:39 pg_stat
drwx------  2 usbmux root   4096  7月 12 09:41 pg_stat_tmp
drwx------  2 usbmux root   4096  7月 12 08:51 pg_subtrans
drwx------  2 usbmux root   4096  7月 12 08:51 pg_tblspc
drwx------  2 usbmux root   4096  7月 12 08:51 pg_twophase
drwx------  3 usbmux root   4096  7月 12 08:51 pg_xlog
-rwx------  1 usbmux netdev   69  7月 12 09:39 postmaster.opts
-rw-------  1 usbmux netdev   67  7月 12 09:39 postmaster.pid
```

### 課題

disposableなコンテナからDockerホストに`data_directory`をもってきたのはよいですが、
全く同じオプションで、新しいコンテナを起動しようとすると、runitの起動スクリプトはユーザーは再作成し、
データベースも作成しようとします。

ユースケースとして、新しいコンテナは`data_directory`のアタッチだけしたいとか考えると、
runitの起動スクリプトは修正しないとちょっと怖いです。

### まとめ

`docker run`の実行時に引数としてデータベース名やユーザー名を渡せて、
コンテナの起動後はすぐにデータベースが使える状態になっているのはうれしいです。

あとはSouthやSequel、Rialsのマイグレートでスキーマを作成して初期データを投入するだけです。

簡単なUIを用意してdocker-pyなどと組み合わせれば、開発用のDBaaSとして十分な気がしてきました。
[docker-dbaas](https://github.com/abulte/docker-dbaas)というもあります。

次はこのPostgreSQLに接続するWebの管理ツール用のコンテナを準備しようと思います。



