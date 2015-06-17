title: "BitTorrent Sync permissions between data volume containers"
date: 2014-11-11 02:50:02
tags:
 - BitTorrentSync
 - DataVolumeContainer
 - CoreOS
 - Vultr
 - IDCFCloud
description: I've been using BitTorrent Sync across data volume containers on CoreOS between Vultr and IDCF Cloud. I wonder why it doen't sync somwetimes. Moreover, I want to set same ownership between containers. I suspect that might be caused by groups settings. Thankfully I found some tutorials. I always check DigitalOcearn's documentation when I am faced with a problem.
---

I've been using BitTorrent Sync across data volume containers on CoreOS between Vultr and IDCF Cloud. I wonder why it doen't sync somwetimes. Moreover, I want to set same ownership on between containers. I suspect that might be caused by groups settings. Thankfully I found some tutorials. I always check DigitalOcearn's documentation when I am faced with a problem.

* [How To Use BitTorrent Sync to Synchronize Directories in Ubuntu 12.04](https://www.digitalocean.com/community/tutorials/how-to-use-bittorrent-sync-to-synchronize-directories-in-ubuntu-12-04)
* [Setting up permissions for bittorrent sync](http://drup.org/setting-permissions-bittorrent-sync)

<!-- more -->

### Permissions and ownership

The `/data` directory is the root shared directory, and the `/data/moin/pages` directory is symlinked to MoinMoin pages directory. A master wiki site is on IDCF Cloud and sync with Vultr instance as backup site. The point is that the root shared directory should be owned by root user which runs BitTorrent Sync in our case. And MoinMoin requires `pages` directory owned by `www-data` which is the user runs Nginx processes in our case too.

``` bash
# hown -R root:www-data /data
# chmod 2775 /data
# chown -R www-data:www-data /data/moin/pages/
# chmod -R g+w /data/moin/pages/
```

### Conclusion

Finally I finished the migration from a CentOS instance on Sakura VPS to a CoreOS cluster on IDCF Cloud and Vultr. Whew! It takes almost one month of my hobby time. This is just the beginning of mastering CoreOS and Docker.