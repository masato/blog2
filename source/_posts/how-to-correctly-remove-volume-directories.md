title: "How to correctly remove volume directories from Docker"
date: 2014-11-05 07:02:41
tags:
 - Docker
 - DataVolumeContainer
 - fleet
 - BitTorrentSync
description: After playing with CoreOS and fleet on Vultr for some weeks, there were too many directories in /var/lib/docker/vfs/dir/ and /var/lib/docker/volumes. I had no choice but to remove these directories manualy. But when it comes to production environment, of course it is not acceptable. Thankfully, I found a blog post of Docker In-depth Volumes. I learned how to correctly remove volume directories with docker rm -v commands.
---

After playing with CoreOS and fleet on Vultr for some weeks, there were too many directories in `/var/lib/docker/vfs/dir/` and `/var/lib/docker/volumes`. I had no choice but to remove these directories manualy. But when it comes to production environment, of course it is not acceptable. Thankfully, I found a blog post of [Docker In-depth: Volumes](http://container42.com/2014/11/03/docker-indepth-volumes/). I learned how to correctly remove volume directories with `docker rm -v` commands.

<!-- more -->

`docker rm` help tells me friendly.

``` bash
$ docker rm --help

Usage: docker rm [OPTIONS] CONTAINER [CONTAINER...]

Remove one or more containers

  -f, --force=false      Force the removal of a running container (uses SIGKILL)
  -l, --link=false       Remove the specified link and not the underlying container
  -v, --volumes=false    Remove the volumes associated with the container
```

### Updating fleet unit files

As usual Docker containers on CoreOS controlled by fleetctl. I create unit files on local machine and submit and start services.

``` bash
$ fleetctl submit system/moin-data@.service
$ fleetctl submit system/moin@.service
$ fleetctl load moin@80.service
$ fleetctl load moin-data@80.service
$ fleetctl start moin@80.service
$ fleetctl start moin-data@80.service
```

During the trial-and-error process, I destroy and start services repeadedly and many garbage volume files area created. 

``` bash
$ fleetctl destroy moin@80.service
$ fleetctl destroy moin-data@80.service
$ fleetctl submit system/moin-data@.service
$ fleetctl submit system/moin@.service
...
```

### BitTorrent Sync strange error

A remained directory which used by BitTorrent Sync data volume container might be cause of `Error while adding folder /data: Selected folder is already added to BitTorrent Sync.` error.

### ExecStartPre directives

I revised my unit files using `docker rm` commands on `ExecStartPre` directive. If using `--volumes-from` flag and binding two containers, both of `docker rm` commands need `-v` flag following [instrucions](http://container42.com/2014/11/03/docker-indepth-volumes/).

``` bash ~/docker_apps/moin-system/moinmoin@.service
[Unit]
Description=MoinMoin Service
Requires=%p-data@%i.service
After=%p-data@%i.service
[Service]
TimeoutStartSec=0
ExecStartPre=-/usr/bin/docker kill %p%i
ExecStartPre=-/usr/bin/docker rm -v %p%i
ExecStartPre=/usr/bin/docker pull masato/moinmoin
ExecStart=/usr/bin/docker run --name %p%i --volumes-from %p-data%i -p 80:80 -p 443:443 210.xxx.xxx.xxx:5000/moin MasatoShimizu
ExecStop=/usr/bin/docker kill %p%i
[Install]
WantedBy=multi-user.target
[X-Fleet]
X-Conflicts=%p@*.service
MachineMetadata=role=moin
```

In a unit file for data volume container I added `-v` flag on `docker rm` commands also.

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
ExecStop=/usr/bin/docker kill %p%i
[Install]
WantedBy=multi-user.target
[X-Fleet]
X-Conflicts=%p@*.service
MachineMetadata=role=moin
```