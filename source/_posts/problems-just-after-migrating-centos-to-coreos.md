title: "Problems just after migrating from CentOS to CoreOS and fleet"
date: 2014-11-12 20:17:58
tags:
 - fleet
 - CoreOS
 - BitTorrent
 - IDCFCloud
 - Vultr
 - uWSGI
description: I finished migrating my wiki site from CentOS to CoreOS yesterday. After shutdown a CentOS instance, it is discovered some problems. It seems that a docker-gen unit and a BitTorrent Sync unit have not been in proper working condition.
---

I finished migrating my wiki site from CentOS to CoreOS yesterday. After shutdown a CentOS instance, it is discovered some problems. It seems that a docker-gen unit and a BitTorrent Sync unit have not been in proper working condition.

<!-- more -->


### May be a central etcd registry problems

I make a etcd be a central registry intentionally, but sometimes a docker host ps states and fleet unit states are in correct.

### BitTorrent synched files permission problems

I make a root shared directory be owned by root user, and set GUID bit. Whenever a systemd timer kicks a sync job between data volume containers, the shared subsidiary directories ownership are reset to root user.

### docker-gen and systemd dependency chain problems

In my fleet unit files relationship, a docker-gen unit shoud be start the second. A docker-gen container suddenly dies the reason why is not known. Then succeeding units all fail.

### Workaround

As a workaround, I stopped a BitTorrent Sync data volume container and systemd timer on IDCF Cloud. After A MoinMoin unit started successfully and a docker-gen generated a nginx.conf, I stopped a docker-gen container on Vultr with fleet. In addition I removed a `die-on-term` frog from uWSGI start command. We have a long way to go.

