title: "MoinMoin in Production on CoreOS - Part5: Data Volume Container"
date: 2014-10-19 23:05:47
tags:
 - MoinMoin
 - CoreOS
 - IDCFCloud
description: Previously I deployed plain MoinMoin CoreOS instance on IDCF Cloud. Next thing should I do is to serve with data volume contaner. Without fleet it is easy to backup and restore using docker-backup. Considering how difficult it is to write fleet unit files for me, it would take time to practice. I think I'm ready to face the day.
---

* `Update 2014-10-20`: [MoinMoin in Production on CoreOS - Part６: Further thinking Data Volume Container](/2014/10/20/docker-moinmoin-idcf-coreos-volumes-further-thinking/)

Previously I deployed plain MoinMoin CoreOS instance on IDCF Cloud. Next thing should I do is to serve with data volume contaner. Without fleet it is easy to backup and restore using [docker-backup](/2014/10/14/docker-data-volume-container-docker-backup/). Considering how difficult it is to write fleet unit files for me, it would take time to practice. I think I'm ready to face the day.

<!-- more -->

### ExecStartPre directive

First thing that comes to my mind is to use ExecStartPre directive. In the ExecStartPre directive it could be started data volume container.

``` bash
ExecStartPre=/usr/bin/docker pull busybox
ExecStartPre=/usr/bin/docker run -v /usr/local/share/moin/data/pages --name wiki-vol busybox true
ExecStart=/usr/bin/docker run --name moin --volumes-from wiki-vol -p 80:80 10.1.1.32:5000/moin
```

I wonder what I should do to restore MoinMoin backuped tar.gz into data volume container?

### moin.service

I prepended many ExecStartPre directives preparing data volume container. Fortunately this attempt was successful.

``` bash ~/docker_apps/moin/moin.service
[Unit]
Description=MoinMoin Service
Requires=docker.service
After=docker.service
[Service]
TimeoutStartSec=0
ExecStartPre=-/usr/bin/docker kill moin wiki-vol
ExecStartPre=-/usr/bin/docker rm moin wiki-vol
ExecStartPre=/usr/bin/wget -N -P /tmp https://www.dropbox.com/s/xxx/141020000501_pages.tar.gz
ExecStartPre=/usr/bin/docker run -v /usr/local/share/moin/data/pages --name wiki-vol busybox true
ExecStartPre=/usr/bin/docker run --rm --volumes-from wiki-vol -v /tmp:/backup ubuntu:14.04 tar zxfv /backup/141020000501_pages.tar.gz --strip-components=1 -C /usr/local/share/moin/data/pages
ExecStart=/usr/bin/docker run --name moin --volumes-from wiki-vol -p 80:80 10.1.1.32:5000/moin
ExecStop=/usr/bin/docker stop moin
[Install]
WantedBy=multi-user.target
```

### Update unit

To update moin unit  I run fleetctl cycle as usual.

``` bash
$ export FLEETCTL_TUNNEL=10.1.1.191
$ fleetctl destroy moin.service
$ fleetctl submit moin.service
$ fleetctl start moin.service
```

Waiting to start moin unit with journal -f. 

``` bash
$ fleetctl journal -f moin.service
```

Finally I opened a MoinMoin URL. 

{% img center /2014/10/19/docker-moinmoin-idcf-coreos-volumes/moin-core-volume.png %}
