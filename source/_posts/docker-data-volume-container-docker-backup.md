title: "MoinMoinをDigitalOceanのCoreOSに移設する - Part3: docker-backupでデータボリュームコンテナのバックアップとリストア"
date: 2014-10-14 00:06:05
tags:
 - DataVolumeContainer
 - docker-backup
 - docker-lloyd
 - MoinMoin
description: WordPressのデータボリュームコンテナを作成したときはcommitしてもvolumeに指定したディレクトリの中身が保存されずに不思議に思いました。MoinMoinのバックアップで初めは挫折したdocker-backupも、ようやくGoのコードを読んで使えるようになりました。データボリュームコンテナの意味がはじめて理解できるようになると、なるほどdisposableコンテナの運用には必須の機能です。
---

* `Update 2014-10-19`: [MoinMoin in Production on CoreOS - Part5: Data Volume Container](/2014/10/19/docker-moinmoin-idcf-coreos-volumes/)
* `Update 2014-10-18`: [MoinMoin in Production on CoreOS - Part4: Staging to IDCF Cloud](/2014/10/18/docker-moinmoin-idcf-coreos/)
* `Update 2014-10-15`: [Docker data volume container backing up with docker-lloyd](/2014/10/15/docker-data-volume-container-docker-lloyd/)

[WordPressのデータボリュームコンテナ](/2014/10/01/docker-wordpress-duplicate-volumes/)を作成したときはcommitしてもvolumeに指定したディレクトリの中身が保存されずに不思議に思いました。[MoinMoinのバックアップ](/2014/10/13/docker-moinmoin-data-volume-container-nginx-uwsgi/)で初めは挫折した[docker-backup](https://github.com/discordianfish/docker-backup)も、ようやくGoのコードを読んで使えるようになりました。

データボリュームコンテナの意味がはじめて理解できるようになると、なるほどdisposableコンテナの運用には必須の機能です。

<!-- more -->


### テンポラリのコンテナを使いバックアップとリストアする

これまで手動でコンテナのボリュームからtar作成してバックアップしていました。基本的にdocker-backupも動作は同じことをします。

``` bash
$ docker run --rm \
             --volumes-from wp \
             -v $(pwd):/backup \
             busybox tar cvf /backup/app_wp_content.tar /app/wp-content
```

データコンテナを作成してuntarしてリストアします。
``` bash
$ docker run --name wp-stg-vol \
             -v /app/wp-content  busybox true
$ docker run --rm --volumes-from wp-stg-vol \
             -v $(pwd):/backup \
             busybox tar xvf /backup/app_wp_content.tar
```

### docker-backupの用意

[docker-backup](https://github.com/discordianfish/docker-backup)のstoreコマンドを使いコンテナにアタッチされているデータボリュームコンテナの中身をアーカイブします。またrestoreコマンドを使うとアーカイブから新しいデータボリュームコンテナを作成することができます。

Goでビルドされた[docker-backup](https://github.com/discordianfish/docker-backup/blob/master/docker-backup.go)コマンドをDockerホストにインストールすることは、[nsenter](/2014/08/03/docker-volume-nsenter-mount/)コマンドのインストールに似ています。
Goのバイナリの場合はそのまま配布しても良いのですが、ローカルで使うコマンドをインストールする方法としてDockerコンテナは環境を汚さないので今後も試していきたいと思います。

docker-backupコンテナをdisposableに使うため、ラッパーのスクリプトを用意します。

``` bash ~/bin/docker-backup
#!/bin/bash
set -eo pipefail
docker run --rm --name docker-backup \
           -v /var/run/docker.sock:/var/run/docker.sock \
           -v /var/lib/docker/vfs/dir:/var/lib/docker/vfs/dir \
           -v $(pwd):/backups \
           fish/docker-backup $@
```

コマンドに実行権限を付けます。

``` bash
$ chmod +x ~/bin/docker-backup
```

~/bin/docker-backupコマンドを使う場合は、カレントのディレクトリをテンポラリのコンテナにマウントしてアーカイブの入出力の場所に使うため、作業ディレクトリを用意します。

``` bash
$ mkdir ~/backups
$ cd !$
```

### docker-backupでバックアップをする

Dockerホストのボリュームディレクトリを確認します。f95fc6d2c9821587c6ca6a2eb6410633e1e7e00a331d1c570ce480a8cbe4d552ディレクトリがバックアップ対象のwikiのデータボリュームです。

``` bash
$ sudo ls -altr /var/lib/docker/vfs/dir/
合計 308
drwx------    3 root     root       4096  9月  8 10:57 ..
drwxr-xr-x    5 root     root       4096 10月  7 23:34 ab53f9e48b4ea05fa262b6ab451c4c0c4150141b2b2a2c2e61b175ccaf390e2e
drwx------    4 root     root       4096 10月 13 09:14 .
drwxr-xr-x 7625 www-data www-data 299008 10月 13 09:14 f95fc6d2c9821587c6ca6a2eb6410633e1e7e00a331d1c570ce480a8cbe4d552
```

wikiのデータボリュームコンテナをinspectして確認します。

``` bash
$ docker inspect --format="&#123;&#123; .Volumes }}" wiki-vol
map[/usr/local/share/moin/data/pages:/var/lib/docker/vfs/dir/f95fc6d2c9821587c6ca6a2eb6410633e1e7e00a331d1c570ce480a8cbe4d552]
```

wikiコンテナは`--volumes-from`でデータボリュームコンテナをマウントしているのでinspectして表示されるボリュームは同じです。

``` bash
$ docker inspect --format="&#123;&#123 .Volumes }}" wiki
map[/usr/local/share/moin/data/pages:/var/lib/docker/vfs/dir/f95fc6d2c9821587c6ca6a2eb6410633e1e7e00a331d1c570ce480a8cbe4d552]
```

storeコマンドを使いwiki-volコンテナをバックアップします。カレントディレクトリがテンポラリコンテナにマウントされた状態のファイル名と、データボリュームコンテナがマウントされているバックアップ対象のコンテナの名前を引数に指定します。

``` bash
$ cd ~/backups
$ docker-backup store /backups/pages.tar wiki
2014/10/13 00:19:39 Storing wiki's volume container as /backups/pages.tar
```

カレントディレクトリにアーカイブが作成されました。

``` bash
$ ls -lh
合計 107M
-rw-r--r-- 1 root root 107M 10月 13 09:19 pages.tar
```

### docker-backupでリストアする

docker-backupはバックアップ前と同じ名前でコンテナを作成します。もとのwiki-volの名前がついたコンテナが存在すると新しく作成できません。

バックアップしたtar.gzの中に、もとのボリュームコンテナ情報を保存したvolume-container.jsonが作成されます。
古いコンテナ名を使って新しいコンテナを作成するため重複してしまいます。コンテナ名を書き換えてリストアできるか今度試してみます。

* [backup.go](https://github.com/discordianfish/docker-backup/blob/master/backup/backup.go#L133)

``` go docker-backup/backup/backup.go
params.Set("name", oldContainer.Name[1:]) // remove leading /
```

リストアする前に起動しているwikiコンテナとwiki-volコンテナを削除します。ステートフルなデータはバックアップしているので、コンテナはdisposableに破棄できます。

``` bash
$ docker kill wiki && docker rm wiki wiki-vol
$ sudo rm -fr /var/lib/docker/vfs/dir/f95fc6d2c9821587c6ca6a2eb6410633e1e7e00a331d1c570ce480a8cbe4d552
```

クリアした状態のボリュームのディレクトリです。wiki-volのf95fc6d2c9821587c6ca6a2eb6410633e1e7e00a331d1c570ce480a8cbe4d552ディレクトリを削除しました。

``` bash
$ sudo ls -altr /var/lib/docker/vfs/dir/
合計 12
drwx------ 3 root root 4096  9月  8 10:57 ..
drwxr-xr-x 5 root root 4096 10月  7 23:34 ab53f9e48b4ea05fa262b6ab451c4c0c4150141b2b2a2c2e61b175ccaf390e2e
drwx------ 3 root root 4096 10月 13 09:24 .
```

restoreコマンドを使いリストアします。storeと同様にカレントディレクトリがテンポラリのコンテナにマウントされた先のファイルパスを指定します。

``` bash
$ cd ~/backups
$ docker-backup restore /backups/pages.tar
2014/10/13 00:25:57 Restoring /backups/pages.tar
```

ボリュームのディレクトリを確認します。新しいデータボリュームコンテナのcf15564ee20b5fb365c277e8d432892fea1d6b2319aab1cb675a477051f7a626ディレクトリが作成されました。

``` bash
$ sudo ls -altr /var/lib/docker/vfs/dir/
合計 308
drwx------    3 root     root       4096  9月  8 10:57 ..
drwxr-xr-x    5 root     root       4096 10月  7 23:34 ab53f9e48b4ea05fa262b6ab451c4c0c4150141b2b2a2c2e61b175ccaf390e2e
drwx------    4 root     root       4096 10月 13 09:25 .
drwxr-xr-x 7625 www-data www-data 299008 10月 13 09:26 cf15564ee20b5fb365c277e8d432892fea1d6b2319aab1cb675a477051f7a626
```

リストアされたwiki-volコンテナをinspectします。

``` bash
$ docker inspect --format="&#123;&#123 .Volumes }}" wiki-vol
map[/usr/local/share/moin/data/pages:/var/lib/docker/vfs/dir/cf15564ee20b5fb365c277e8d432892fea1d6b2319aab1cb675a477051f7a626]
```

wikiコンテナをリストされたwiki-volを`--volumes-from`に指定して新しくrunします。

``` bash
$ docker run --name wiki -d --volumes-from wiki-vol masato/moinmoin
```

wikiコンテナをinspectすると新しいデータボリュームコンテナがマウントされています。

``` bash
$ docker inspect --format="&#123;&#123 .Volumes }}" wiki
map[/usr/local/share/moin/data/pages:/var/lib/docker/vfs/dir/cf15564ee20b5fb365c277e8d432892fea1d6b2319aab1cb675a477051f7a626]
```

wikiコンテナのIPアドレスを確認してブラウザで開くと、リストア前の状態に戻っています。

```
$ docker inspect --format="&#123;&#123 .NetworkSettings.IPAddress }}" wiki
172.17.0.240
```
