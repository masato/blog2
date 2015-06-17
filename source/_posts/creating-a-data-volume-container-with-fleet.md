title: "Creating a data volume container with fleet"
date: 2014-11-03 16:23:24
tags:
 - DataVolumeContainer
 - BitTorrentSync
 - CoreOS
 - Vultr
 - MoinMoin
 - fleet
description: I'm going to  separate some MoinMoin stateful data into a data volume container. In this way it can be done to maintain pages data individually. Going stateless is always good habits for Docker Way.
---

I'm going to separate some MoinMoin stateful data into a data volume container. In this way it can be done to maintain pages data individually. Going stateless is always good habits for Docker Way.

<!-- more -->

### fleet unit files

A moin@.service requires a separated moin-data@.service unit, it will fail if a data volume container is not exists.

``` bash ~/docker_apps/moin/system/moin@.service
[Unit]
Description=MoinMoin Service
Requires=%p-data@%i.service
After=%p-data@%i.service

[Service]
TimeoutStartSec=0
ExecStartPre=-/usr/bin/docker kill %p%i
ExecStartPre=-/usr/bin/docker rm -v %p%i
ExecStartPre=/usr/bin/docker pull 210.xxx.xxx.xxx:5000/moin
ExecStart=/usr/bin/docker run --name %p%i --volumes-from %p-data%i -p 80:80 -p 443:443 210.xxx.xxx.xxx:5000/moin MasatoShimizu
ExecStop=/usr/bin/docker stop %p%i

[Install]
WantedBy=multi-user.target

[X-Fleet]
X-Conflicts=%p@*.service
MachineMetadata=role=moin
```

A moin-data@.service unit should be running before moin@.service. I do not ensure a BitTorrent Sync process is finished before running a moin@.service. I will fix this issue some time later.

``` bash ~/docker_apps/moin/system/moin-data@.service
[Unit]
Description=MoinMoin Data Service
Requires=docker.service
After=docker.service

[Service]
TimeoutStartSec=0
ExecStartPre=-/usr/bin/docker kill %p%i
ExecStartPre=-/usr/bin/docker rm -v %p%i
ExecStartPre=-/usr/bin/docker pull masato/btsync
ExecStart=/usr/bin/docker run --name %p%i masato/btsync xxx
ExecStop=/usr/bin/docker stop %p%i

[Install]
WantedBy=multi-user.target

[X-Fleet]
X-Conflicts=%p@*.service
MachineMetadata=role=moin

```

### Retoring MoinMoin stateful data

This this script downloads the latest backup file from Dropbox and restores into BitTorrent Sync powered data volume container. The original plan was to schedule restoring jobs through cron every four hours. Owing to a strange befavior of sync and time consuming process, I stopped a cron job. From this time, I separate stateful date such as user accounts and SSL Certificate from MoinMoin container step by step.

``` bash ~/bin/moin-update
#!/bin/bash
set -eo pipefail

BACKUP_DIR=moin_backup

cd /tmp

LATEST_FILE=$($HOME/bin/dropbox_uploader.sh list /$BACKUP_DIR  | tail -n +2 | awk '{print $3}' | sort -nr | head -1)
$HOME/bin/dropbox_uploader.sh download /$BACKUP_DIR/$LATEST_FILE

docker run --rm --volumes-from moin-data -v $(pwd):/backup ubuntu /bin/bash -c "rm -fr /data/pages/* && mkdir -p /data/pages && tar zxf /backup/$LATEST_FILE --strip-components=1 -C /data/pages/ && chown -R 33:33 /data/pages"
docker run --rm --volumes-from moin-data -v $HOME/docker_apps/moin:/backup busybox sh -c 'mkdir -p /data/user && cp /backup/user /data/user/##########.##.##### && chown -R 33:33 /data/user'
rm -f /tmp/$LATEST_FILE
```
