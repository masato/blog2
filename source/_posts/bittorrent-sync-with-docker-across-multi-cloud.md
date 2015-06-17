title: "BitTorrent Sync with Docker across multi-cloud"
date: 2014-11-01 16:44:26
tags:
 - BitTorrentSync
 - MoinMoin
 - Vultr
 - IDCFCloud
description: After a while using ctlc/btsync to sync a MoinMoin between Vultr and IDCF Cloud instances, it suddenly stopped syncing. The verion of Bittorrent Sync in the current ctlc/btsync image is 1.3.105 and it is a little old.  I don't know whether the old version is related, I intend to write another Dockerfile and bebug sync processes as practice.
---


After a while using [ctlc/btsync](https://registry.hub.docker.com/u/ctlc/btsync/) to sync between Vultr and IDCF Cloud instances, it suddenly stopped syncing. The verion of Bittorrent Sync in a current ctlc/btsync image is 1.3.105 and it is a little old. I don't know whether the old version is related, I intend to write another Dockerfile and bebug sync processes as practice.

<!-- more -->

### A BitTorrent Sync Image

I created an automated build [repository](https://registry.hub.docker.com/u/masato/btsync/). It's convenient for me the default locale is Japanese when I check the sync progress by timestamp.

``` bash ~/docker_apps/btsync/Dockerfile
FROM ubuntu:14.04
MAINTAINER Masato Shimizu <ma6ato@gmail.com>

ENV LANG ja_JP.UTF-8
RUN locale-gen $LANG && update-locale $LANG && \
    ln -sf /usr/share/zoneinfo/Asia/Tokyo /etc/localtime

ADD http://download-lb.utorrent.com/endpoint/btsync/os/linux-x64/track/stable /btsync.tar.gz
RUN tar xzf /btsync.tar.gz -C /usr/bin && \
    rm /btsync.tar.gz && mkdir -p /btsync/.sync

EXPOSE 55555
ADD run.sh /run.sh

VOLUME ["/data"]
ENTRYPOINT ["/run.sh"]
```

In the run.sh script file making sure an old process is not existed after restarting container.

``` bash ~/docker_apps/btsync/run.sh
#!/bin/bash
set -eo pipefail

SECRET=${@:-`btsync --generate-secret`}

echo "Starting btsync with secret: $SECRET"

rm -f /btsync/btsync.pid

[ ! -f /btsync/btsync.conf ] && cat > /btsync/btsync.conf <<EOF
{
  "device_name": "Sync Server",
  "listening_port": 55555,
  "storage_path": "/btsync/.sync",
  "pid_file": "/btsync/btsync.pid",
  "check_for_updates": false,
  "use_upnp": false,
  "download_limit": 0,
  "upload_limit": 0,
  "shared_folders": [{
    "secret": "$SECRET",
    "dir": "/data",
    "use_relay_server": true,
    "use_tracker": true,
    "use_dht": false,
    "search_lan": true,
    "use_sync_trash": false
  }]
}
EOF

btsync --config /btsync/btsync.conf --nodaemon
```