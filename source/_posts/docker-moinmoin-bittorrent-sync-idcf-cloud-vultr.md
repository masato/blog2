title: "MoinMoin in Production on CoreOS - Part10: BitTorrent Sync between IDCF Cloud and Vultr"
date: 2014-10-27 22:08:35
tags:
 - CoreOS
 - DataVolumeContainer
 - BitTorrentSync
 - MoinMoin
 - Vultr
 - IDCFCloud
 - DropboxUploader
 - fleet
description: I'm making plans to migrate my MoinMoin from Sakura VPS to Vultr. I could mostly done those preparations. One of the remaining works is to backup MoinMoin pages data and sync data between multi-clouds. I decited to use a syncing technology of BitTorrent Sync following a CenturyLink Labs tutorial. My first goal is to sync data between Vultr and IDCF Cloud both are sit on CoreOS.
---

I'm making plans to migrate my MoinMoin from Sakura VPS to Vultr. I could mostly done those preparations. One of the remaining works is to backup MoinMoin pages data and sync data between multi-clouds. I decited to use a syncing technology of [BitTorrent Sync](http://www.getsync.com/) following a [CenturyLink Labs tutorial](http://www.centurylinklabs.com/persistent-distributed-filesystems-in-docker-without-nfs-or-gluster/). My first goal is to sync data between Vultr and IDCF Cloud both are sit on CoreOS.

<!-- more -->

### 1st IDCF cloud instance using DropboxUploader

The first thing to do is to run a BitTorrent Sync client container from [ctlc/btsync](https://registry.hub.docker.com/u/ctlc/btsync/) image. After finihsed running a container, it is provided a secret key to use in all of the sync containers. The secret key can be got via a `docker logs` command.

``` bash
$ docker run -d --name moin-btsync ctlc/btsync
$ docker logs moin-btsync
Starting btsync with secret: xxx
```

I write a command file using [DropboxUploader](https://github.com/andreafabrizi/Dropbox-Uploader). In this script, I download the latest backuped MoinMoin archive from Dropbox and untar to BitTorrent Sync directory.

``` bash ~/bin/moin-update
#!/usr/bin/env bash
set -eo pipefail
BACKUP_DIR=moin_backup
cd /tmp
LATEST_FILE=$(dropbox_uploader.sh list /$BACKUP_DIR  | tail -n +2 | awk '{print $3}' | sort -nr | head -1)
dropbox_uploader.sh download /$BACKUP_DIR/$LATEST_FILE
docker run -it --rm --volumes-from moin-btsync ubuntu rm -fr /data/*
docker run --rm --volumes-from moin-btsync -v $(pwd):/backup ubuntu tar zxvf /backup/$LATEST_FILE --strip-components=1 -C /data
rm /tmp/$LATEST_FILE
```

### 2nd Vultr instance via moin@.service

Another BitTorrent Sync container is deployed to Vulter via fleet unit file. I change data volume container for MoinMoin from my own image to ctlc/btsync image.

``` bash ~/docker_apps/moin/moin@.service
[Unit]
Description=MoinMoin Service
Requires=docker.service
After=docker.service

[Service]
TimeoutStartSec=0
ExecStartPre=-/usr/bin/docker kill %p%i
ExecStartPre=-/usr/bin/docker rm %p%i

ExecStartPre=/bin/bash -c '/usr/bin/docker start moin-vol || /usr/bin/docker run -d --name moin-vol ctlc/btsync xxx'
ExecStartPre=/usr/bin/docker pull xxx.ngrok.com/moin

ExecStart=/usr/bin/docker run --name %p%i --volumes-from moin-vol -p 80:80 -p 443:443 xxx.ngrok.com/moin
ExecStop=/usr/bin/docker stop %p%i

[Install]
WantedBy=multi-user.target

[X-Fleet]
X-Conflicts=%p@*.service
MachineMetadata=role=moin
```

I revise the Dockerfile as data volume to `/data` directory adjusting to `ctlc/btsync` container. In addition, this version of Nginx is configured with SSL, although using s self-signed  certificate.

``` bash ~/docker_apps/moin/Dockerfile
FROM ubuntu:14.04
MAINTAINER Masato Shimizu <ma6ato@gmail.com>
ENV HOME /root
ENV WORKDIR /root

RUN sed -i~ -e 's/archive.ubuntu.com/ftp.jaist.ac.jp/' /etc/apt/sources.list && \
    apt-get update && \
    DEBIAN_FRONTEND=noninteractive apt-get install -qy language-pack-ja

ENV LANG ja_JP.UTF-8
RUN locale-gen $LANG && update-locale $LANG && \
    ln -sf /usr/share/zoneinfo/Asia/Tokyo /etc/localtime

RUN DEBIAN_FRONTEND=noninteractive \
  apt-get install -qy nginx uwsgi uwsgi-plugin-python python wget unzip openssl
## OpenSSL
RUN cd /etc/ssl \
 && openssl genrsa -out ssl.key 4096 \
 && openssl req -new -batch -key ssl.key -out ssl.csr \
 && openssl x509 -req -days 3650 -in ssl.csr -signkey ssl.key -out ssl.crt
ADD nginx.conf /etc/nginx/sites-available/default
RUN wget http://static.moinmo.in/files/moin-1.9.7.tar.gz -P /tmp && \
    tar xzf /tmp/moin-1.9.7.tar.gz -C /tmp && \
    cd /tmp/moin-1.9.7 && \
    python setup.py install --force --prefix=/usr/local && \
    sed -e '/sitename/s/Untitled //' \
        -e '/page_front_page.*Front/s/#\(page_front_page\)/\1/' \
        -e '/superuser/ { s/#\(superuser\)/\1/; s/YourName/ScottTiger/ }' \
        -e '/page_front_page/s/#u/u/' \
        -e '/acl_rights_before/s/.*/    acl_rights_default = u"ScottTiger:read,write,delete,revert,admin"\n&/'\
        -e '/theme_default/s/modernized/europython/' \
        -e '/theme_default/s/.*/    tz_offset = 9.0\n&/' \
       /usr/local/share/moin/config/wikiconfig.py > \
       /usr/local/share/moin/wikiconfig.py && \
       chown -Rh www-data. /usr/local/share/moin/data \
                           /usr/local/share/moin/underlay && \
    wget https://bitbucket.org/thesheep/moin-europython/get/3995afe116b0.zip -P /tmp && \
    unzip /tmp/3995afe116b0.zip -d /tmp && \
    cd /tmp/thesheep* && \
    cp europython.py /usr/local/share/moin/data/plugin/theme && \
    mkdir /usr/local/lib/python2.7/dist-packages/MoinMoin/web/static/htdocs/europython && \
    cp -R css /usr/local/lib/python2.7/dist-packages/MoinMoin/web/static/htdocs/europython && \
    cp -R img /usr/local/lib/python2.7/dist-packages/MoinMoin/web/static/htdocs/europython
ADD screen.css /usr/local/lib/python2.7/dist-packages/MoinMoin/web/static/htdocs/europython/css/screen.css
ADD common.css /usr/local/lib/python2.7/dist-packages/MoinMoin/web/static/htdocs/europython/css/common.css
ADD user /usr/local/share/moin/data/user/1338720549.58.37773
ADD run.sh /run.sh
RUN mkdir /data && chown www-data:www-data /data && \
    mv /usr/local/share/moin/data/pages /usr/local/share/moin/data/pages.orig && \
    ln -s /data /usr/local/share/moin/data/pages
VOLUME ["/data"]
CMD ["/bin/bash", "/run.sh"]
RUN apt-get clean && rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*
```

And the `run.sh` is revised also. In the first place of this script, the symlinked `/data` directory's ownership is changed to `www-data`. 

``` bash ~/docker_apps/moin/run.sh
#!/bin/bash

chown -R www-data:www-data /data
uwsgi --uid www-data \
    -s /tmp/moin.sock \
    --plugins python \
    --pidfile /var/run/uwsgi-moinmoin.pid \
    --wsgi-file server/moin.wsgi \
    -M -p 4 \
    --chdir /usr/local/share/moin \
    --python-path /usr/local/share/moin \
    --harakiri 30 \
    --die-on-term \
    --daemonize /var/log/uwsgi/app/moinmoin.log
nginx -g 'daemon off;'
```