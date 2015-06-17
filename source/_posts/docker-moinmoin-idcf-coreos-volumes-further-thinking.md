title: "MoinMoin in Production on CoreOS - Part6: Further thinking Data Volume Container"
date: 2014-10-20 07:01:40
tags:
 - MoinMoin
 - CoreOS
 - IDCFCloud
description: To deal with a data volume contaner in fleet unit file on staging IDCF Cloud, my first attempt was very naive. It colud work just incase it was the first time. So I rethinked building data volume container. I don't understand this is correct way, but it works when there exists a data volume container already and using that container.
---

To deal with a data volume contaner in fleet unit file on staging IDCF Cloud, my first attempt was very naive. It colud work just incase it was the first time. So I rethinked building data volume container. I don't understand this is correct way, but it works when there exists a data volume container already and using that container.

<!-- more -->

### Building data volume container image

I want to keep fleet unit files concise. Before understanding well something lack of simplicity is an obstacle to study. I created data volume contaner's Dockerfile. The pourpose of this image is to download pages archive from Dropbox and untar to volume directory beforehand.

``` bash ~/docker_apps/moin-data/Dockerfile
FROM ubuntu:14.04
MAINTAINER Masato Shimizu <ma6ato@gmail.com>
RUN sed -i~ -e 's/archive.ubuntu.com/ftp.jaist.ac.jp/' /etc/apt/sources.list && \
    apt-get update && apt-get install -y wget
RUN mkdir -p /usr/local/share/moin/data && \
    wget -P /tmp https://www.dropbox.com/s/xxx/141020080501_pages.tar.gz && \
    tar zxfv /tmp/141020080501_pages.tar.gz  -C /usr/local/share/moin/data/
VOLUME ["/usr/local/share/moin/data/pages"]
CMD ["/bin/sh","true"]
RUN apt-get clean && rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*
```
 
And push to local docker-registry

``` bash
$ docker build -t masato/moin-data .
$ docker tag masato/moinmoin localhost:5000/moin
$ docker push localhost:5000/moin
```

### Fleet unit file

Then I revised my moin.service file. Through trial and error, I check named data volume container is exists in ExecStartPre directive. If it eixsts then keep it , or does not then run container pulling from docker-registry.

``` bash ~/docker_apps/moin/moin.service
[Unit]
Description=MoinMoin Service
Requires=docker.service
After=docker.service
[Service]
TimeoutStartSec=0
ExecStartPre=-/usr/bin/docker kill moin
ExecStartPre=-/usr/bin/docker rm moin
ExecStartPre=/bin/bash -c '/usr/bin/docker inspect moin-vol >/dev/null 2>&1 || /usr/bin/docker run -v /\
usr/local/share/moin/data/pages --name moin-vol 10.1.1.32:5000/moin-data true && true'
ExecStart=/usr/bin/docker run --name moin --volumes-from moin-vol -p 80:80 10.1.1.32:5000/moin
ExecStop=/usr/bin/docker stop -f moin
[Install]
WantedBy=multi-user.target
```

### Update unit

After updating moin unit file, even so, moin.service is still working.

``` bash
$ export FLEETCTL_TUNNEL=10.1.1.191
$ fleetctl destroy moin.service
$ fleetctl submit moin.service
$ fleetctl start moin.service
$ fleetctl journal -f moin.service
```