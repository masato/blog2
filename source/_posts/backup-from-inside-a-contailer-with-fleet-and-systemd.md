title: "Backup from inside a container with fleet and systemd"
date: 2014-11-08 13:21:28
tags:
 - CoreOS
 - fleet
 - systemd
 - docker-enter
 - Vultr
description: A migration of my MoinMoin instance from Sakura VPS to CoreOS on Vultr is almost completed. A remaining thing is no doubt a backup planning. I used to backup wiki pages to Dropbox on a CentOS instance. It's very easy to schedule a backup job with shell script and crontab. But it's suddenly complicated in regard to a Docker, fleet/systemd and CoreOS combined environment.
---

A migration of my MoinMoin instance from Sakura VPS to CoreOS on Vultr is almost completed. A remaining thing is no doubt a backup planning. I used to backup wiki pages to Dropbox on a CentOS instance. It's very easy to schedule a backup job with shell script and crontab . But it's suddenly complicated in regard to a Docker, fleet/systemd and CoreOS combined environment.

<!-- more -->

### Creating timestamp environments

It's a little tricky to create environments in systemd unit files. I created some environments  both of on `Environment` and `ExecStart` directives successfully. It was not worked when I used a `date` command for creating timestamp on `ExecStart` directive. In addition, it should be escaped with `%%` in place of `%` to [specify a single percent sign](http://www.freedesktop.org/software/systemd/man/systemd.unit.html#Specifiers).

``` bash
Environment="now=$(/bin/date +%%Y-%%m-%%d-%%H-%%M-%%S)"
ExecStart=/bin/bash -c "\
  backup_dir=/tmp/moin/${now} && \
```

### docker-enter on CoreOS

CoreOS 444.5.0 comes with nsenter, but [docker-enter](https://github.com/jpetazzo/nsenter/blob/master/docker-enter) is not installed. We install docker-enter via cloud-config.yml to CoreOS. docker-enter is a small shell script which let a program to execute inside the namespace.

``` bash
  /opt/bin/docker-enter $container PYTHONPATH=/usr/local/share/moin /usr/local/bin/moin maint reducewiki --target-dir=$backup_dir && \
```

In addition, it also could be possible to output a stream to a host file.

``` bash
  /opt/bin/docker-enter $container tar czf - -C $backup_dir pages > $file_name && \
```

### Scripting in ExecStart directive

Using a backslash '\' character for line continuation makes a long shell script readable. Don't for get suffix a backslash on the very last statement.

``` bash
ExecStart=/bin/bash -c "\
  backup_dir=/tmp/moin/${now} && \
  file_name=/tmp/${now}_pages.tar.gz && \
  container=moinmoin%i && \
  echo backup to: $file_name && \
  /opt/bin/docker-enter $container mkdir -p $backup_dir && \
  /opt/bin/docker-enter $container PYTHONPATH=/usr/local/share/moin /usr/local/bin/moin maint reducewiki --target-dir=$backup_dir && \
  /opt/bin/docker-enter $container tar czf - -C $backup_dir pages > $file_name && \
  /opt/bin/docker-enter $container rm -fr $backup_dir && \
  echo success  : $file_name \
"
```

### A complete fleet unit file

This is a fleet unit file for a MoinMoin Service. The next thing to do is to execute backup unit with a sytemd timers. Before that, we will confirm this unit file work alone.

``` bash ~/docker_apps/moinmoin-system/moinmoin-backup@.service
[Unit]
Description=MoinMoin Backup Service
Requires=moinmoin@%i.service
After=moinmoin@%i.service

[Service]
TimeoutStartSec=0
Environment="now=$(/bin/date +%%Y-%%m-%%d-%%H-%%M-%%S)"
ExecStart=/bin/bash -c "\
  backup_dir=/tmp/moin/${now} && \
  file_name=/tmp/${now}_pages.tar.gz && \
  container=moinmoin%i && \
  echo backup to: $file_name && \
  /opt/bin/docker-enter $container mkdir -p $backup_dir && \
  /opt/bin/docker-enter $container PYTHONPATH=/usr/local/share/moin /usr/local/bin/moin maint reducewiki --target-dir=$backup_dir && \
  /opt/bin/docker-enter $container tar czf - -C $backup_dir pages > $file_name && \
  /opt/bin/docker-enter $container rm -fr $backup_dir && \
  echo success  : $file_name \
"

[Install]
WantedBy=multi-user.target

[X-Fleet]
Conflicts=%p@*.service
MachineMetadata=role=moin
```

### Start fleet unit files

A correct way to update fleet unit files is a little bit cumbersome. It should be complied to a shell script, oddly enough a batch execution could not be successful every time for now.

``` bash
$ fleetctl destroy moinmoin-backup@.service
$ fleetctl destroy moinmoin-backup@80.service
$ fleetctl submit moinmoin-backup@.service
$ fleetctl load moinmoin-backup@80.service
$ fleetctl start moinmoin-backup@80.service
```

Then we confirm a unit is running by showing journal logs.

``` bash
$ fleetctl journal -f  moinmoin-backup@80.service
Nov 08 14:14:22 localhost systemd[1]: Starting MoinMoin Backup Service...
Nov 08 14:14:22 localhost systemd[1]: Started MoinMoin Backup Service.
Nov 08 14:14:22 localhost bash[962]: backup to: /tmp/2014-11-08-14-14-22_pages.tar.gz
Nov 08 14:14:22 localhost bash[962]: 2014-11-08 14:14:22,338 INFO MoinMoin.log:151 using logging configuration read from built-in fallback in MoinMoin.log module
Nov 08 14:14:22 localhost bash[962]: 2014-11-08 14:14:22,338 INFO MoinMoin.log:157 Running MoinMoin 1.9.8 release code from /usr/local/lib/python2.7/dist-packages/MoinMoin
Nov 08 14:14:22 localhost bash[962]: 2014-11-08 14:14:22,476 INFO MoinMoin.config.multiconfig:127 using wiki config: /usr/local/share/moin/wikiconfig.pyc
Nov 08 14:14:34 localhost bash[962]: success : /tmp/2014-11-08-14-14-22_pages.tar.gz
```
