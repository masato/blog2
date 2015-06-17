title: "Docker data volume container backing up with docker-lloyd"
date: 2014-10-15 00:06:05
tags:
 - MoinMoin
 - docker-lloyd
 - docker-backup
 - IDCFObjectStorage
description: I forked docker-lloyd repo for using with IDC Frontier Object Storage and store image in masato/docker-lloyd. It' helpful for me to do scheduling backup statefull containers.
---

I forked [docker-lloyd](https://github.com/masato/docker-lloyd) repo for using with IDC Frontier Object Storage and store image in [masato/docker-lloyd](https://registry.hub.docker.com/u/masato/docker-lloyd/). It' helpful for me to do scheduling backup statefull containers.

<!-- more -->

### Backup Container

``` bash
$ cd ~/docker_apps
$ git clone git@github.com:masato/docker-lloyd.git
```

I defined .s3cfg explicitly for using modified IDCF's endpoint.  

``` bash run
     s3cmd --access_key="$ACCESS_KEY" --secret_key="$SECRET_KEY" \
#          -c /dev/null $S3CMD_OPTS put "$BACKUPS/$1.tar.gz" $BUCKET
           -c /.s3cfg $S3CMD_OPTS put "$BACKUPS/$1.tar.gz" $BUCKET
     rm "$BACKUPS/$1.tar.gz"
```

This .s3cfg has IDCF specific host name.

``` bash .s3cfg
#host_base = s3.amazonaws.com
#host_bucket = %(bucket)s.s3.amazonaws.com
host_base = ds.jp-east.idcfcloud.com
host_bucket = %(bucket)s.ds.jp-east.idcfcloud.com
```

### How to use

First of all I created bucket for storing data container archives.

``` bash
$ s3cmd -c ~/.s3cfg.idcf mb s3://moinmoin
Bucket 's3://moinmoin/' created
```

Pulling forked image from Docker Hub Registry and run disposable container. 

``` bash
$ docker pull masato/docker-lloyd
$ docker run --rm --name docker-lloyd \
         -v /var/run/docker.sock:/docker.sock \
         -v /var/lib/docker/vfs/dir:/var/lib/docker/vfs/dir \
         -e ACCESS_KEY={ACCESS_KEY} \
         -e SECRET_KEY={SECRET_KEY} \
         masato/docker-lloyd \
         s3://moinmoin wiki
[=] wiki
wiki: Backup
2014/10/13 03:03:52 Storing wiki's volume container as /backups/wiki.tar
wiki: Upload
File '/backups/wiki.tar.gz' stored as 's3://moinmoin/wiki.tar.gz' (53636927 bytes in 8.8 seconds, 5.81 MB/s) [1 of 1]
wiki: Sleep
```

Confirming inside of bucket.

``` bash
$ s3cmd -c ~/.s3cfg.idcf ls s3://moinmoin
2014-10-13 01:23  53635965   s3://moinmoin/wiki.tar.gz
```

### Restore Container 

Beforehand restoring container, I cleard existing containers.

``` bash
$ docker kill wiki && docker rm wiki wiki-vol
$ sudo rm -rf /var/lib/docker/vfs/dir/cf15564ee20b5fb365c277e8d432892fea1d6b2319aab1cb675a477051f7a626
```

and download backuped archive using s3cmd. 

``` bash
$ cd ~/backups
$ s3cmd -c ~/.s3cfg.idcf get s3://moinmoin/wiki.tar.gz
s3://moinmoin/wiki.tar.gz -> ./wiki.tar.gz  [1 of 1]
```

Gzipping tar.gz and restoring data volume container from tar file.

``` bash
$ gzip -d wiki.tar.gz
$ docker-backup restore /backups/wiki.tar
2014/10/13 03:21:49 Restoring /backups/wiki.tar
$ docker inspect --format="&#123;&#123; .Volumes }}" wiki-vol
map[/usr/local/share/moin/data/pages:/var/lib/docker/vfs/dir/b0090c53517478b9ab0feb319c61501f1109dd502236eb0421f6871d12723485]
```

Finally run MoinMoin image mounting restored data volume container.

``` bash
$ docker run --name wiki -d --volumes-from wiki-vol masato/moinmoin
```
