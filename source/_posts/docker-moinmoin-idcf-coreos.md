title: "MoinMoin in Production on CoreOS - Part4: Staging to IDCF Cloud"
date: 2014-10-18 03:14:05
tags:
 - MoinMoin
 - CoreOS
 - IDCFCloud
description: I backuped MoinMoin pages archives to IDCF Object Storage. Before deploying to the production environment on DigitalOcean, I prepared CoreOS instance on IDCF Cloud for staging environment. This is very first time I wrote MoinMoin installing as a fleet unit file. 
---

* `Update 2014-10-20`: [MoinMoin in Production on CoreOS - Part６: Further thinking Data Volume Container](/2014/10/20/docker-moinmoin-idcf-coreos-volumes-further-thinking/)
* `Update 2014-10-19`: [MoinMoin in Production on CoreOS - Part5: Data Volume Container](/2014/10/19/docker-moinmoin-idcf-coreos-volumes/)

I backuped MoinMoin pages archives to IDCF Object Storage. Before deploying to the production environment on DigitalOcean, I prepared CoreOS instance on IDCF Cloud for staging environment. This is very first time I wrote MoinMoin installing as a fleet unit file. 


<!-- more -->

### CoreOS from ISO

At first I changed password of coreos user from IDCF Cloud browser console. This enable us to ssh with password.

``` bash
$ sudo passwd core
```

Then I could login with pasword via SSH. 

``` bash
$ ssh core@10.1.1.191 -o PreferredAuthentications=password
```

### Edit Cloud-config file

The first step to setting up a new CoreOS cluster is generating a new discovery URL, a unique address that stores peer CoreOS addresses and metadata. The easiest way to do this is to use

I executed curl command for generating new discovery URL.

``` bash
$ curl -w "\n" https://discovery.etcd.io/new
https://discovery.etcd.io/485b242d1cb90f50ada691ee2f37a6c7
```

This is my usual cloud-config.yml to create a basic CoreOS instance.

``` yml ~/cloud-config.yml
#cloud-config
coreos:
  etcd:
    discovery: https://discovery.etcd.io/485b242d1cb90f50ada691ee2f37a6c7
    addr: $private_ipv4:4001
    peer-addr: $private_ipv4:7001
  fleet:
    public-ip: $private_ipv4  
  units:
    - name: etcd.service
      command: start
    - name: fleet.service
      command: start
    - name: timezone.service
      command: start
      content: |
        [Unit]
        Description=timezone
        [Service]
        Type=oneshot
        RemainAfterExit=yes
        ExecStart=/usr/bin/ln -sf ../usr/share/zoneinfo/Japan /etc/localtime
write_files:
  - path: /etc/profile.d/nse-function.sh
    permissions: 0755
    content: |
      function nse() {
        sudo nsenter --pid --uts --mount --ipc --net --target $(docker inspect --format="&#123;&#123; .State.Pid }}" $1)
      }
  - path: /etc/environment
    permissions: 0644
    content: |
      COREOS_PUBLIC_IPV4=
      COREOS_PRIVATE_IPV4=10.1.1.191
  - path: /etc/profile.d/nse-function.sh
    permissions: 0755
    content: |
      function nse() {
        sudo nsenter --pid --uts --mount --ipc --net --target $(docker inspect --format="&#123;&#123; .State.Pid }}" $1)
      }
ssh_authorized_keys:
  - ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDaMqAk6OeBLkyU6kj9HHq18xIBgSewSzZDRT3dYRnE92F5tz/16W/McxQ3XoIu8Z7aDNt5WEE+vWatAAItUPWQLRKjM0zwPC04hciMXloC7WjEeIikksg9hGzuc6nals7sfItCOx1aKqLFnEpRmXnzmKycYT8YqRBFzbGFJ+RSGmskVJeMaKUXunF43JH93ugmMCFqv92OnUHsY0tNeqI/EEncghHIKRwoHq46dNvECsvhX8ZruscQB3oGK24+sjZlL78kXeFXM4q3ZbefZESF4iKhs3BMfmT7ihNnuntdZhAHupBxg5npolw6yGZLjZcilduSnuTR6BDWtNEIS0S1 deis
```

Since I created this CoreOS instance from ISO the stable version is bumped to 444.5.0. It is needed to be set -V option for the latest version.

``` bash
$ sudo coreos-install -d /dev/sda -V 444.5.0 -C stable -c ./cloud-config.yml
...
Installing cloud-config...
Success! CoreOS stable 444.5.0 is installed on /dev/sda
```

Restarting to finish installing.
 
``` bash
$ sudo reboot
```

### Using fleetctl

Before using fleetctl command from outside of fleet cluster, it shoud be set `FLEETCTL_TUNNEL` environment pointing to one of the fleet machines. It may also required to start ssh-agent beforehand.

``` bash
$ export FLEETCTL_TUNNEL=10.1.1.191
$ fleetctl list-machines
MACHINE         IP              METADATA
4a236dee...     10.1.1.191      -
```

This is a basic fleet unit file just starting docker container safelly. I will add data volume container unit next time.

``` bash ~/docker_apps/moin/moin.service
[Unit]
Description=MoinMoin Service
Requires=docker.service
After=docker.service
[Service]
TimeoutStartSec=0
ExecStartPre=-/usr/bin/docker kill moin
ExecStartPre=-/usr/bin/docker rm moin
ExecStartPre=/usr/bin/docker pull 10.1.1.32:5000/moin
ExecStart=/usr/bin/docker run --name moin -p 80:80 10.1.1.32:5000/moin
ExecStop=/usr/bin/docker kill moin
[Install]
WantedBy=multi-user.target
```

After finished writing moin.service unit file, I submit and start moin unit using fleetctl command.

``` bash
$ fleetctl submit moin.service
$ fleetctl start moin.service
Unit moin.service launched on 4a236dee.../10.1.1.191
$ fleetctl list-unit-files
UNIT            HASH    DSTATE          STATE           TARGET
moin.service    b665615 launched        launched        4a236dee.../10.1.1.191
$ fleetctl journal -f moin.service
```

I confirmed moin.service unit running.

``` bash
$ fleetctl list-units
UNIT            MACHINE                 ACTIVE  SUB
moin.service    4a236dee.../10.1.1.191  active  running
```

And go to this URL.

{% img center /2014/10/18/docker-moinmoin-idcf-coreos/moin-core.png %}


It is also possible to login to target machine passing unit name.

``` bash
$ fleetctl ssh moin
CoreOS (stable)
```

