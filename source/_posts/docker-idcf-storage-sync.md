title: "Using Docker for WordPress backup to IDCF Object Storage"
date: 2014-10-31 23:36:00
tags:
 - s3cmd
 - IDCFCloud
 - IDCFObjectStorage
 - Docker
 - WordPress
---

I am using [BackUpWordPress](https://wordpress.org/plugins/backupwordpress/) for my WordPress blog backup solution. This plugin can create schedules for backup. I make a schedule to backup twice a day. Of course, since I am running my WordPress site on Docker, I set up a cron job putting archives to IDCF Object Storage on Docker. I created my repository on Docker Hub Registry.

<!-- more -->

### masato/idcf-storage-sync

For a Docker Hub's Automated Build Repository, I set a [project](https://github.com/masato/idcf-storage-sync) on GitHub. The Dockerfile is below.

``` bash Dockerfile
FROM ubuntu:14.04.1
MAINTAINER Masato Shimizu <ma6ato@gmail.com>
ENV LANG ja_JP.UTF-8
RUN locale-gen ja_JP.UTF-8 && \
    update-locale LANG=ja_JP.UTF-8 && \
    ln -sf /usr/share/zoneinfo/Asia/Tokyo /etc/localtime
RUN apt-get update && \
    DEBIAN_FRONTEND=noninteractive apt-get install -qy \
    python-setuptools python-dateutil python-magic git
RUN git clone https://github.com/s3tools/s3cmd.git /s3cmd && \
    cd /s3cmd && python setup.py install
ADD run.sh /run.sh
ADD s3cfg /.s3cfg
CMD ["/run.sh"]
RUN apt-get clean && rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*
```

And this is a run.sh which actually runs s3cmd sync. 

``` bash
#!/bin/bash
set -eo pipefail
function die() {
  echo >&2 "$@"
  exit 1
}
function s3sync()
{
  s3cmd --config=/.s3cfg --access_key="$ACCESS_KEY" --secret_key="$SECRET_KEY" sync $S3CMD_OPTS "$SRC_DIR" "$BUCKET_NAME"
}
test -n "$BUCKET_NAME" || die "Please set BUCKET_NAME environment variable"
test -n "$ACCESS_KEY" -a -n "$SECRET_KEY" || die "Please set ACCESS_KEY and SECRET_KEY environment variables"
test -n "SRC_DIR" || die "Please set SRC_DIR environment variable"
s3sync
```

Then I create a helper shell script file to execute `docker run` command.

``` bash ~/bin/wp-backup
#!/bin/bash
set -eo pipefail

docker run --rm --name idcf-storage-sync  \
  -e ACCESS_KEY={ACCESS_KEY}   \
  -e SECRET_KEY={SECRET_KEY} \
  -e BUCKET_NAME=s3://{BUCKET_NAME}   \
  -e SRC_DIR={backup_directory}   \
  -e S3CMD_OPTS="--exclude=* --include=*.zip --delete-removed" \
  --volumes-from {data_volume_container}  \
  masato/idcf-storage-sync >> /tmp/wp-backup.log
exit 0
```

I edit a crontab file to run this script.

``` bash
0 8,20 * * * ~/bin/wp-backup > /dev/null 2>&1
```



