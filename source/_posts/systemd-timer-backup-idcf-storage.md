title: "Using systemd timers for backup with IDCF Object Storage"
date: 2014-11-09 22:50:52
tags:
 - IDCFObjectStorage
 - MoinMoin
 - s3cmd
 - systemd
 - fleet
description: My wiki pages were originally planned to be backuped to Dropbox or Google Drive which have user friendly interface and inherently folder synchronization. These cloud storage services use OAuth 2.0 authentication for API. It's not a problem for operations. But in a disposable environment such as Docker, unfortunately it is difficult to generate an access token and set it to containers interactively. That is why I switched to IDCF Object Storage for my backup plan.
---

My wiki pages were originally planned to be backuped to Dropbox or Google Drive which have user friendly interface and inherently folder synchronization. These cloud storage services use OAuth 2.0 authentication for API. It's not a problem for operations. But in a disposable environment such as Docker, unfortunately it is difficult to generate an access token and set it to containers interactively. That is why I switched to IDCF Object Storage for my backup plan.

<!-- more -->

### Backup to IDCF Object Storage

I recently created a [idcf-storage-sync](https://registry.hub.docker.com/u/masato/idcf-storage-sync/) Docker image for this purpose. It's a simple wrapper of s3cmd sync based on Docker container.

### Systemd Timers

We use a systemd timer as an replacement of crontab. A `OnUnitActiveSec` directive is used for executing a unit which is defined at `Unit` directive every four hours.

``` bash  ~/docker_apps/moinmoin-system/moinmoin-backup@.timer
[Unit]
Description=MoinMoin Every four hours Timer
[Timer]
OnUnitActiveSec=4hour
Unit=%p@%i.service
[Install]
WantedBy=timers.target
[X-Fleet]
MachineOf=%p@%i.service
```

### MoinMoin Backup Service

I use a etcd cluster as a central repository for avoiding complex probrems of leader election. So it might be a ordinary use of etcdctl, we should get values from etcd and set to environment variables before ExecStart directive in my environment.

``` bash  ~/docker_apps/moinmoin-system/moinmoin-backup@.service
[Unit]
Description=MoinMoin Backup Service
Requires=moinmoin@%i.service
After=moinmoin@%i.service
[Service]
TimeoutStartSec=0
EnvironmentFile=/etc/environment
Environment="ACCESS_KEY=$(etcdctl --no-sync get /idcf/storage/access_key)"
Environment="SECRET_KEY=$(etcdctl --no-sync get /idcf/storage/secret_key)"
Environment="BUCKET_NAME=s3://$(etcdctl --no-sync get /idcf/storage/bucket/moinmoin)"
Environment="now=$(/bin/date +%%Y-%%m-%%d-%%H-%%M-%%S)"
Environment="dest_dir=/opt/moin/backups"
ExecStartPre=/usr/bin/docker pull masato/idcf-storage-sync
ExecStartPre=/bin/bash -c "mkdir -p $dest_dir"
ExecStart=/bin/bash -c "\
  tmp_dir=/tmp/moin/${now} && \
  file_name=${dest_dir}/${now}_pages.tar.gz && \
  container=moinmoin%i && \
  /opt/bin/docker-enter $container mkdir -p $tmp_dir && \
  /opt/bin/docker-enter $container PYTHONPATH=/usr/local/share/moin /usr/local/bin/moin maint reduc\
ewiki --target-dir=$tmp_dir && \
  /opt/bin/docker-enter $container tar czf - -C $tmp_dir pages > $file_name && \
  /opt/bin/docker-enter $container rm -fr $tmp_dir && \
  echo success  : $file_name && \
  docker run --rm --name idcf-storage-sync  \
    -e ACCESS_KEY=${ACCESS_KEY}  \
    -e SECRET_KEY=${SECRET_KEY} \
    -e BUCKET_NAME=${BUCKET_NAME} \
    -e SRC_DIR=/backups \
    -e S3CMD_OPTS='--delete-removed' \
    -v $dest_dir:/backups \
    masato/idcf-storage-sync && \
  find $dest_dir -ctime +2 -exec rm {} \; \
"
[Install]
WantedBy=multi-user.target
[X-Fleet]
Conflicts=%p@*.service
MachineMetadata=role=moin
```

A reduced wiki pages archive is saved in a CoreOS host directory. We could sync files in the directory and IDCF Object Storage's bucket using `s3cmd sync` via idcf-storage-sync container. We set a `--delete-removed` flag for letting s3cmd remove any files not in the local directory. And the last line would clean up old files before two days.

### Set timer for a target service

We submit and load a target unit file and timer unit file. And we only start a timer unit for scheduling. 

``` bash
$ fleetctl submit moinmoin-backup@.service
$ fleetctl load moinmoin-backup@80.service
$ fleetctl submit moinmoin-backup@.timer
$ fleetctl load moinmoin-backup@80.timer
$ fleetctl start moinmoin-backup@80.timer
```

### Debug timers

For confirmation we set a timer to run every 5 minitus. 

``` bash  ~/docker_apps/moinmoin-system/moinmoin-backup@.timer
[Unit]
Description=MoinMoin Every five minutes Timer
[Timer]
OnUnitActiveSec=5min
Unit=%p@%i.service
[Install]
WantedBy=timers.target
[X-Fleet]
MachineOf=%p@%i.service
```

`fleetctl status` shows that moinmoin-backup@80.service is inactive, because we didn't start.

``` bash
$ fleetctl status moinmoin-backup@80.service
● moinmoin-backup@80.service - MoinMoin Backup Service
   Loaded: loaded (/run/fleet/units/moinmoin-backup@80.service; enabled)
   Active: inactive (dead) since Mon 2014-11-09 13:29:27 JST; 1min 58s ago
```

As showing a journal log, the target service has been executed by systemd timer every 5 minutes.

``` bash
$ fleetctl journal -f moinmoin-backup@80.service
...
Nov 10 13:29:11 localhost bash[11168]: success : /opt/moin/backups/2014-11-10-13-28-59_pages.tar.gz
Nov 10 13:29:27 localhost bash[11168]: File '/backups/2014-11-10-13-28-59_pages.tar.gz' stored as 's3://xxx/backups/2014-11-10-13-28-59_pages.tar.gz' (54340493 bytes in 12.2 seconds, 4.25 MB/s) [1 of 1]
Nov 10 13:29:27 localhost bash[11168]: Done. Uploaded 54340493 bytes in 12.2 seconds, 4.24 MB/s.  Copied 0 files saving 0 bytes transfer.
...
Nov 10 13:34:13 localhost bash[11423]: success : /opt/moin/backups/2014-11-10-13-33-59_pages.tar.gz
Nov 10 13:34:28 localhost bash[11423]: File '/backups/2014-11-10-13-33-59_pages.tar.gz' stored as 's3://xxx/backups/2014-11-10-13-33-59_pages.tar.gz' (54340216 bytes in 11.7 seconds, 4.44 MB/s) [1 of 1]
Nov 10 13:34:28 localhost bash[11423]: Done. Uploaded 54340216 bytes in 11.7 seconds, 4.43 MB/s.  Copied 0 files saving 0 bytes transfer.
```

