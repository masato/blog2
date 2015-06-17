title: "Installing Google Drive CLI on CoreOS"
date: 2014-10-28 23:23:10
tags:
 - GoogleDrive
 - Go
 - fleet
 - CoreOS
 - CLI
description: I'm thinking about installing command line tools to CoreOS via fleet unit files. This idea comes from Kubernetes Installation on CoreOS. As usual to install binaries written in Go is very easy, just put it somewhere in your path. Combining with fleet oneshot service, I'm going to install a Google Drive CLI client on CoreOS.
---

I'm thinking about installing command line tools to CoreOS via fleet unit files. This idea comes from Kubernetes Installation on CoreOS. As usual to install binaries written in Go is very easy, just put it somewhere in your path. Combining with fleet oneshot service, I'm going to install a Google Drive CLI client on CoreOS.

<!-- more -->

### Google Drive CLI install

[gdrive](https://github.com/prasmussen/gdrive) is a command line utility for Google Drive. Thankfully this tool is written in Go, the installation is just saving a  binary to a location in your path. For CoreOS I download a [drive-linux-amd64 v1.3.0](https://drive.google.com/uc?id=0B3X9GlR6EmbnTjk4MGNEbEFRRWs) binary.

### Oneshot fleet unit file

I prepare a fleet unit file for installing gdrive command file. This file is described as `Type=oneshot` and `RemainAfterExit=yes` in `[Service]` directive.  The definitions are suitable for command installation.

``` bash ~/docker_apps/moin/gdrive.service
[Unit]
Description=Google Drive Service

[Service]
ExecStartPre=-/usr/bin/mkdir -p /opt/bin
ExecStartPre=/usr/bin/curl -L -o /opt/bin/gdrive https://drive.google.com/uc?id=0B3X9GlR6EmbnTjk4MGNEbEFRRWs
ExecStart=/usr/bin/chmod +x /opt/bin/gdrive
RemainAfterExit=yes
Type=oneshot

[X-Fleet]
Global=true
```

And start a gdrive.service fleet unit.

``` bash
$ fleetctl submit gdrive.service
$ fleetctl start gdrive.service
```

### Using gdrive command

For the first time of running gdrive command it requires a OAuth certificate. Following the instruction I open the url in you browser and get the verification code.

``` bash
$ gdrive
Go to the following link in your browser:
Enter verification code:
...
```

Basic usage examples are in [gdrive's README](https://github.com/prasmussen/gdrive).

``` bash
drive [global options] <verb> [verb options]
```