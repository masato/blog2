title: "How to correctly configure CoreOS iptables on Vultr"
date: 2014-11-07 23:21:05
tags:
 - iptables
 - CoreOS
 - Vultr
description: I have been puzzled by Docker iptables on CoreOS. While reading a official Network Configuration page, I think I understand the reason. 
---

I have been [puzzled](/2014/11/04/difficult-docker-networking-configuration/) by Docker iptables on CoreOS. While reading a official [Network Configuration](https://docs.docker.com/articles/networking/) page, I think I understand the reason. 

<!-- more -->

### Be published directly

The problem is that I published a container's port to the host. That's why the container is exposed to the world.

``` bash ~/docker_apps/moinmoin-system/moinmoin@.service
[Unit]
Description=MoinMoin Service
Requires=%p-data@%i.service
After=%p-data@%i.service

[Service]
TimeoutStartSec=0
ExecStartPre=-/usr/bin/docker kill %p%i
ExecStartPre=-/usr/bin/docker rm -v %p%i
ExecStartPre=/usr/bin/docker pull masato/moinmoin
ExecStart=/usr/bin/docker run --name %p%i \
  --volumes-from %p-data%i \
  -p 80:80 \
  masato/moinmoin \
  AdminName
ExecStop=/usr/bin/docker kill %p%i

[Install]
WantedBy=multi-user.target

[X-Fleet]
X-Conflicts=%p@*.service
MachineMetadata=role=moin
```

### Not be published directly

If a container is not published a port, it seems that Docker does not set DNAT rules to the container, althogh it is not accessible from outside of the host. If some applications in a container shoud be published to the world, there is a need with help of a reverse proxy. I will write about this later.

``` bash ~/docker_apps/moinmoin-system/moinmoin@.service
[Unit]
Description=MoinMoin Service
Requires=%p-data@%i.service
After=%p-data@%i.service

[Service]
TimeoutStartSec=0
ExecStartPre=-/usr/bin/docker kill %p%i
ExecStartPre=-/usr/bin/docker rm -v %p%i
ExecStartPre=/usr/bin/docker pull masato/moinmoin
ExecStart=/usr/bin/docker run --name %p%i \
  --volumes-from %p-data%i \
  -e VIRTUAL_HOST=www.example.com \
  -e SSL=true \
  masato/moinmoin AdminName
ExecStop=/usr/bin/docker kill %p%i

[Install]
WantedBy=multi-user.target

[X-Fleet]
X-Conflicts=%p@*.service
MachineMetadata=role=moin
```