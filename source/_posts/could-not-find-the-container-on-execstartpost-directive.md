title: "Could not find the container on ExecStartPost directive"
date: 2014-11-06 21:06:09
tags:
 - DataVolumeContainer
 - fleet
description: Just having a try, I want to create a index.html file through binding from another container on ExecStartPost directive. When I run a plain Nginx container for testing, I don't want to prepare some data volume container or modified images for Nginx images beforehand. Using with ExecStartPost directives, I thought that it might be possible to get it, but actually I couldn't.
---

Just having a try, I want to create a index.html file through binding from another container on ExecStartPost directive. When I run a plain Nginx container for testing, I don't want to prepare some data volume container or modified images for Nginx images beforehand. Using with ExecStartPost directives, I thought that it might be possible to get it, but actually I couldn't.

<!-- more -->

### My trash unit file

I executed `docker run` from a trusted [dockerfile/nginx](https://registry.hub.docker.com/u/dockerfile/nginx/) image. I added `-v /var/www/html` flag and binded from another container on ExecStart directive.

On ExecStartPost directive 

``` bash ~/docker_apps/moin/system/nginx-minimal@.service
[Unit]
Description=Nginx Minimal Service
Requires=docker.service
After=docker.service
[Service]
TimeoutStartSec=0
ExecStartPre=-/usr/bin/docker kill %p%i
ExecStartPre=-/usr/bin/docker rm -v %p%i
ExecStartPre=/usr/bin/docker pull dockerfile/nginx
ExecStartPre=/usr/bin/docker pull busybox
ExecStart=/usr/bin/docker run --name %p%i \
  -v /var/www/html \
  -p 80:80 \
  dockerfile/nginx
ExecStartPost=/usr/bin/docker run --rm \
  --volumes-from %p%i \
  busybox \
  sh -c "echo 'Hello World' > /var/www/html/index.html"
ExecStop=/usr/bin/docker stop %p%i
[X-Fleet]
X-Conflicts=%p@*.service
MachineMetadata=role=moin
```

It seems that on ExecStartPost directive it could not be possible to find the container and to mount its volumes which creating by ExecStart.  This causes whole unit start process to fail.

``` bash
$ fleetctl submit system/nginx-minimal@.service
$ fleetct load nginx-minimal@80.service
$ fleetct start nginx-minimal@80.service
$ fleetctl journal -f nginx-minimal@80.servece
...
Nov 06 21:58:22 localhost systemd[1]: Starting Nginx Minimal Service...
Nov 06 21:58:22 localhost docker[32496]: Error response from daemon: No such container: nginx-minimal80
Nov 06 21:58:22 localhost docker[32496]: 2014/11/07 10:58:22 Error: failed to kill one or more containers
Nov 06 21:58:22 localhost docker[32506]: Error response from daemon: No such container: nginx-minimal80
Nov 06 21:58:22 localhost docker[32506]: 2014/11/07 10:58:22 Error: failed to remove one or more containers
Nov 06 21:58:22 localhost docker[32521]: Pulling repository dockerfile/nginx
Nov 06 21:58:25 localhost docker[32536]: Pulling repository busybox
Nov 06 21:58:28 localhost systemd[1]: nginx-minimal@80.service: control process exited, code=exited status=1
Nov 06 21:58:28 localhost docker[32552]: 2014/11/07 10:58:28 Error response from daemon: Cannot start container 960cdce5ff448377aa5ef9c134f9fcd111715bfbe185c4fb2c86a2a240252114: Container nginx-minimal80 not found. Impossible to mount its volumes
Nov 06 21:58:28 localhost docker[32574]: Error response from daemon: No such container: nginx-minimal80
Nov 06 21:58:28 localhost docker[32574]: 2014/11/07 10:58:28 Error: failed to stop one or more containers
Nov 06 21:58:28 localhost systemd[1]: nginx-minimal@80.service: control process exited, code=exited status=1
Nov 06 21:58:28 localhost systemd[1]: nginx-minimal@80.service: main process exited, code=exited, status=2/INVALIDARGUMENT
Nov 06 21:58:28 localhost systemd[1]: Failed to start Nginx Minimal Service.
Nov 06 21:58:28 localhost systemd[1]: Unit nginx-minimal@80.service entered failed state.
```

### It works by mounting a host directory 

* `Update 2014-11-08`

After two days I came up with that it could be possible by mounting a host directory on ExecStartPre directive. It works this time. 

``` bash ~/docker_apps/moinmoin-system/nginx-minimal@.service
[Unit]
Description=Nginx Minimal Service
Requires=docker.service
After=docker.service
[Service]
TimeoutStartSec=0
ExecStartPre=-/usr/bin/docker kill %p%i
ExecStartPre=-/usr/bin/docker rm -v %p%i
ExecStartPre=/usr/bin/docker pull dockerfile/nginx
ExecStartPre=/bin/sh -c "mkdir -p /opt/nginx/html; echo 'Hello World' > /opt/nginx/html/index.html"
ExecStart=/usr/bin/docker run --name %p%i \
  -v /opt/nginx/html:/var/www/html \
  -p 80:80 \
  dockerfile/nginx
ExecStop=/usr/bin/docker kill %p%i
[X-Fleet]
X-Conflicts=%p@*.service
MachineMetadata=role=moin
```