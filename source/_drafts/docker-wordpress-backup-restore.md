title: docker-wordpress-backup-restore
tags:
---


## 構成

* wp-docker-fsv: プロダクション環境、バックアップ元のDockerホスト
* salt-minion: 検証環境、リストア先のDockerホスト

## はじめに

WordPressのアップグレードやプラグインの検証のため、プロダクション環境をバックアップして、検証環境でリストアする手順をまとめます。

WordPressコンテナは、commitしてプライベートのdocker-registryにpushして使います。`/app/wp-content`のコンテンツはtarにしてコピー後、新しいデータボリュームコンテナをマウントします。

MySQLコンテナは新規コンテナを起動します。`/var/lib/mysql`のデータはtarにしてコピー後、新しいデータボリュームコンテナをマウントします。

## 作業ディレクトリの作成

マルチプレクサなどを使ってシェルを2つ起動して、バックアップ元とリストア先のDockerホストにそれぞれSSHで接続します。

wp-docker-fsvにログインして作業ディレクトリを作成します。

``` bash
$ ssh -A mshimizu@wp-docker-fsv
$ mkdir ~/blog-back/0115
$ cd !$
```

salt-minionにログインして作業ディレクトリを作成します。

``` bash
$ ssh -A mshimizu@salt-minion
$ mkdir ~/blog-back/0115
$ cd !$
```

### プロダクション環境のDocker

プロダクション環境のwp-docker-fsvのコンテナを一覧します。db-prod-volとwp-prod-volはデータボリュームコンテナなので停止させています。

``` bash
$ docker ps
CONTAINER ID        IMAGE                                           COMMAND               CREATED             STATUS                    PORTS                NAMES
8a08b8cf215c        210.129.196.12:5000/prod-engineer-blog:latest   "/run-wordpress.sh"   10 weeks ago        Up 10 weeks               0.0.0.0:80->80/tcp   wp-prod
61320f1cdb70        tutum/mysql:5.6                                 "/run.sh"             10 weeks ago        Up 10 weeks               3306/tcp             db-prod
40ef91dd756d        busybox:latest                                  "true"                10 weeks ago        Exited (0) 10 weeks ago                        db-prod-vol
0e47477f8823        busybox:latest                                  "true"                10 weeks ago        Exited (0) 10 weeks ago                        wp-prod-vol
```

Dockerのバージョンを確認します。

``` bash
$ docker -v
Docker version 1.3.0, build c78088f
```

wp-docker-fsvの作業ディレクトリに移動します。

``` bash
$ mkdir cd ~/blog-back/0115
$ cd !$
```


## MySQLデータボリュームコンテナ(db-prod-vol)のバックアップ



MySQLデータボリュームコンテナをバックアップします。

``` bash
$ docker run --rm --volumes-from db-prod-vol -v $(pwd):/backup busybox tar cvf /backup/var_lib_mysql.tar /var/lib/mysql
```

バックアップしたtarをリストア先のsalt-minionにscpで転送します。

``` bash
$ scp var_lib_mysql.tar salt-minion:blog-back/0115/
```

## MySQLコンテナ(db-prod)のバックアップ


MySQLコンテナをcommitして、docker-registryにpushします。

``` bash
$ docker commit db-prod idcf/stg-engineer-blog-db
$ docker tag  idcf/stg-engineer-blog-db 210.129.196.12:5000/stg-engineer-blog-db
$ docker push 210.129.196.12:5000/stg-engineer-blog-db
```

## WordPressデータボリュームコンテナ(wp-prod-vol)のバックアップ

WordPressデータボリュームコンテナをバックアップします。

``` bash
$ docker run --rm --volumes-from wp-prod-vol -v $(pwd):/backup busybox tar cvf /backup/app_wp_content.tar /app/wp-content
```

バックアップしたtarをリストア先のsalt-minionにscpで転送します。

``` bash
$ scp app_wp_content.tar salt-minion:blog-back/0115/
```


## WordPressコンテナ(wp-prod)のバックアップ

WordPressコンテナをcommitして、docker-registryにpushします。

``` bash
$ docker commit wp-prod idcf/stg-engineer-blog
$ docker tag  idcf/stg-engineer-blog 210.129.196.12:5000/stg-engineer-blog
$ docker push 210.129.196.12:5000/stg-engineer-blog
```


### 検証環境のDocker

salt-minionvの作業ディレクトリに移動します。

``` bash
$ mkdir cd ~/blog-back/0115
$ cd !$
```

Dockerのバージョンを確認します。プロダクション環境と同じビルド番号です。

``` bash
$ docker -v
Docker version 1.3.0, build c78088f
```


## MySQLデータボリュームコンテナ(db-stg2-vol)のリストア

MySQLデータボリュームコンテナ(db-stg2-vol)を作成し、`/var/lib/mysql`にuntarします。

``` bash
$ cd ~/blog-back/0115
$ docker run --name db-stg2-vol -v /var/lib/mysql  busybox true
$ docker run --rm --volumes-from db-stg2-vol -v $(pwd):/backup busybox tar xvf /backup/var_lib_mysql.tar
```

## MySQLデータコンテナ(db-stg2)の作成

MySQLコンテナ(db-stg2)は新規に起動します。

``` bash
$ docker run --name db-stg2 -d --volumes-from db-stg2-vol tutum/mysql:5.6
```

db-stg2をinspectします。

``` bash
$ DB_HOST=$(docker inspect --format="&#123;&#123; .NetworkSettings.IPAddress }}" db-stg2)
$ echo $DB_HOST
172.17.0.28
```

## WordPressデータボリュームコンテナ(wp-stg2-vol)のリストア

WordPressデータボリュームコンテナ(wp-stg2-vol)を作成し、app_wp_content.tarをリストアします。

``` bash
$ cd ~/blog-back/0115
$ docker run --name wp-stg2-vol -v /app/wp-content  busybox true
$ docker run --rm --volumes-from wp-stg2-vol -v $(pwd):/backup busybox tar xvf /backup/app_wp_content.tar
```

## WordPressコンテナ(wp-stg2)のリストア

プロダクション環境からdocker-registryにpushしたWordPressのDockerイメージをpullします。

``` bash
$ docker pull 210.129.196.12:5000/stg-engineer-blog
```

WordPressコンテナ(wp-stg2)を起動します。

``` bash
$ DB_HOST=$(docker inspect --format="&#123;&#123; .NetworkSettings.IPAddress }}" db-stg2)
$ docker run -d \
  --name wp-stg2 \
  --link db-stg2:db \
  -e DB_HOST="${DB_HOST}" \
  -e DB_PORT="3306" \
  -e DB_PASS="xxx" \
  -e DB_USER="admin" \
  -e DB_NAME="wordpress" \
  --volumes-from wp-stg2-vol \
  -p 8086:80 \
  210.129.196.12:5000/stg-engineer-blog
```

## MySQLの起動確認

nse(nsenterのラッパー)コマンドを使い、MySQLコンテナに直接アタッチしてデータベースの起動確認をします。

``` bash
$ nse db-stg2
root@a43ac5f0ee77:/# mysql wordpress
mysql> exit
root@a43ac5f0ee77:/# exit
```

## WordPressの起動確認 (まだ起動していない)

ベースにしているPHPのDockerイメージにバグがあり、まだイメージをビルドし直していないのでワークアラウンドします。pidが残っているため、WordPressの起動に失敗します。

``` bash
$ docker logs wp-stg2
AH00558: apache2: Could not reliably determine the server's fully qualified domain name, using 172.17.0.31. Set the 'ServerName' directive globally to suppress this message
httpd (pid 1) already running
$ docker ps -a | grep wp-stg2
da99b723aee7        210.129.196.12:5000/stg-engineer-blog:latest    "/run-wordpress.sh"    4 minutes ago       Exited (0) 4 minutes ago                                           wp-stg2


inspectして、起動に失敗しているaufsのIDを確認します。

``` bash
$ docker inspect wp-stg2
    "HostnamePath": "/var/lib/docker/containers/aa457425fff4ad5477399fe01b1c1ad3d6317283cb65b0ae422ba77adb818940/hostname",
    "HostsPath": "/var/lib/docker/containers/aa457425fff4ad5477399fe01b1c1ad3d6317283cb65b0ae422ba77adb818940/hosts",
    "Id": "aa457425fff4ad5477399fe01b1c1ad3d6317283cb65b0ae422ba77adb818940",
    "Image": "bad1eddc9daa074f5d810db2467eb61ddc515f8f55033e979bd581921f199cc1",
...
```

該当のaufsからApacheのpidを探して消します。

``` bash
$ sudo find /var/lib/docker/aufs -type f -name "apache2.pid"
/var/lib/docker/aufs/diff/bad1eddc9daa074f5d810db2467eb61ddc515f8f55033e979bd581921f199cc1/run/apache2/apache2.pid
$ sudo rm /var/lib/docker/aufs/diff/bad1eddc9daa074f5d810db2467eb61ddc515f8f55033e979bd581921f199cc1/run/apache2/apache2.pid
```

WordPressコンテナ(wp-stg2)をstartして、MySQLコンテナへの接続を確認します。

``` bash
$ docker start  wp-stg2
$ nse wp-stg2
$ mysql -uadmin -pxxx -h 172.17.0.28 wordpress
mysql> 
```


## システムの起動確認

psを確認します。MySQLとWordPrssのコンテナが正常に起動しました。

``` bash
$ docker ps
CONTAINER ID        IMAGE                                           COMMAND               CREATED             STATUS              PORTS                                   NAMES
aa457425fff4        210.129.196.12:5000/stg-engineer-blog:latest    "/run-wordpress.sh"   14 minutes ago      Up 3 minutes        0.0.0.0:8086->80/tcp                    wp-stg2
a43ac5f0ee77        tutum/mysql:5.6                                 "/run.sh"             37 minutes ago      Up 37 minutes       3306/tcp                                db-stg2
```

ブラウザから確認します。

http://210.140.149.189:8086/blog/


## 管理画面のアクセス制御

最後に管理画面のアクセス制御をします。プロダクション環境と`WP_*`環境変数が異なります。WordPressコンテナにアタッチして検証環境にあわせて修正します。

``` bash 
$ nse wp-stg2
$ vi /app/wp-config.php
$white_addr = array("210.168.36.157","158.205.104.203");
if (in_array( $_SERVER['REMOTE_ADDR'], $white_addr)) {
  define('WP_SITEURL', 'http://210.140.149.189:8086/blog');
  define('WP_HOME', 'http://210.140.149.189:8086/blog');
  define('WP_CONTENT_URL', 'http://210.140.149.189:8086/blog/wp-content');
...
```

WordPressコンテナのプロセスにHUPを送って再起動させます。

```
$ docker kill -s HUP wp-stg2
```

管理画面をブラウザから確認します。 

http://210.140.149.189:8086/blog/wp-admin
