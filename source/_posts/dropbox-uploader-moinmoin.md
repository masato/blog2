title: 'MoinMoinのpagesをDropboxにバックアップする'
date: 2014-04-29 22:35:14
tags:
 - Dropbox
 - MoinMoin
 - DropboxUploader
description: MoinMoinのpagesは、定期的にAmazonS3とDropboxにバックアップしています。CentOS6.5上でDropbox Uploaderをcron実行します。
---

* `Update 2014-10-13`: [MoinMoinをDigitalOceanのCoreOSに移設する - Part2: Dockerイメージの作成](/2014/10/13/docker-moinmoin-data-volume-container-nginx-uwsgi/)
* `Update 2014-10-12`: [MoinMoinをDigitalOceanのCoreOSに移設する - Part1: はじめに](/2014/10/12/docker-moinmoin-design/)

MoinMoinのpagesは、定期的にAmazonS3とDropboxにバックアップしています。
CentOS6.5上で[Dropbox Uploader](http://www.andreafabrizi.it/?dropbox_uploader)をcron実行します。

<!-- more -->

### moin_uploader.sh

MoinMoinのpagesは、reducewikiをして履歴を削除してからアーカイブします。

``` bash ~/bin/moin_uploader.sh
#!/bin/bash
BACKUP_DIR=moin_backup
TODAY=`date +%y%m%d%H%M%S`
TMP_MOIN=/tmp/dropboxmoin
FILENAME="$TODAY"_pages.tar.gz
TARGZ_PATH=$BACKUP_DIR/$FILENAME
rm -rf $TMP_MOIN
mkdir -p $BACKUP_DIR $TMP_MOIN
export PYTHONPATH=/usr/local/share/moin:/usr/local/lib/python2.6/site-packages
/usr/local/bin/moin maint reducewiki --target-dir=$TMP_MOIN
tar cfz $HOME/$TARGZ_PATH -C $TMP_MOIN/ pages
$HOME/bin/dropbox_uploader.sh upload $HOME/$TARGZ_PATH /$TARGZ_PATH
rm $HOME/$TARGZ_PATH
exit 0
```
パーミッションを設定します。

``` bash
$ chmod u+x ~/bin/moin_uploader.sh
```

### cron

4時間間隔で、MoinMoinのpagesをDropboxにアップロードします。

``` bash
$ crontab -e
5 */4 * * * ~/bin/moin_uploader.sh > /dev/null 2>&1
```
