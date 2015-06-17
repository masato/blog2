title: "MoinMoin in Production on CoreOS - Part9: Booting with iPXE on Vultr"
date: 2014-10-26 17:46:17
tags:
 - CoreOS
 - fleet
 - MoinMoin
 - Vultr
 - DigitalOcean
 - IDCFCloud
 - iPXE
 - ngrok
 - iptables
description: I've finally gotten used to creating CoreOS cluster with fleet. After successfully multi-cloud deployment across IDCF Cloud and DigitalOcean, this time I want to let a Vultr instance join to my CoreOS cluster and run a MoinMoin fleet unit service. It should be possible to be cloud provider agnostic using CoreOS as a platform of infrastructure.
---

I've finally gotten used to creating CoreOS cluster with fleet. After successfully multi-cloud deployment across IDCF Cloud and DigitalOcean, this time I want to let a Vultr instance join to my CoreOS cluster and run a MoinMoin fleet unit service. It should be possible to be cloud provider agnostic using CoreOS as a platform of infrastructure.

<!-- more -->

### Serve a iPXE ChainURL

To run a CoreOS instance on Vultr via iPXE I refer to following official tutorials.

* [Booting CoreOS via iPXE](https://coreos.com/docs/running-coreos/bare-metal/booting-with-ipxe/)
* [Running CoreOS on a Vultr VPS](https://coreos.com/docs/running-coreos/cloud-providers/vultr/)

First of all I create a script with iPXE command lines. 

``` txt ~/vultr/script.txt
#!ipxe

set base-url http://stable.release.core-os.net/amd64-usr/current
kernel ${base-url}/coreos_production_pxe.vmlinuz sshkey="ssh-rsa AAAA..."
initrd ${base-url}/coreos_production_pxe_image.cpio.gz
boot
```

The iPXE script file is served by a Python SimpleHTTPServer.

``` bash
$ cd ~/vultr
$ python -m SimpleHTTPServer 3000
Serving HTTP on 0.0.0.0 port 3000 ...
```

### Creating instance on Vultr

* Server Type: PERFORMANCE SERIES
* Location: Asia Tokyo 
* Operating System: Custom
* iPXE ChainURL:  http://210.xxx.xxx.xxx:3000/script.txt
* Server Size: 1 CPU 768MB MEMORY

It takes about one minute to run a CoreOS instance.

### Install CoreOS

Unlike DigitalOcean's CoreOS integration it needed to take another steps. Because a iPXE booted CoreOS is running from RAM it should be installed to disk, otherwise it is useless. It's similar to create CoreOS instance on IDCF Cloud. I ssh login to the just created CoreOS instance on Vultr.

``` bash
$ ssh -A core@108.61.162.139
```

I create cloud-config.yml file pointing to dedicated etcd endpoint. As with the DigitalOCean instance I edit iptables rules in write_files directive.

``` yml ~/cloud-config.yml
#cloud-config
coreos:
  fleet:
    etcd_servers: http://210.xxx.xxx.xxx:4001
    metadata: role=moin
  units:
    - name: etcd.service
      mask: true
    - name: fleet.service
      command: start
    - name: timezone.service
      command: start
      content: |
        [Unit]
        Description=Set the timezone

        [Service]
        ExecStart=/usr/bin/timedatectl set-timezone Asia/Tokyo
        RemainAfterExit=yes
        Type=oneshot
    - name: iptables.service
      command: start
      content: |
        [Unit]
        Description=Packet Filtering Framework
        DefaultDependencies=no
        After=systemd-sysctl.service
        Before=sysinit.target
        [Service]
        Type=oneshot
        ExecStart=/usr/sbin/iptables-restore /etc/iptables.rules
        ExecReload=/usr/sbin/iptables-restore /etc/iptables.rules
        ExecStop=/usr/sbin/iptables --flush
        RemainAfterExit=yes
        [Install]
        WantedBy=multi-user.target
write_files:
  - path: /etc/environment
    content: |
      COREOS_PUBLIC_IPV4=$public_ipv4
      COREOS_PRIVATE_IPV4=$private_ipv4
  - path: /opt/bin/nse
    permissions: 0755
    content: |
      #!/bin/sh
      
      sudo nsenter --pid --uts --mount --ipc --net --target $(docker inspect --format="&#123;&#123; .State.Pid }}" $1)
  - path: /opt/bin/docker-enter
    permissions: 0755
    content: |
      #!/bin/sh

      if [ -e $(dirname "$0")/nsenter ]; then
          # with boot2docker, nsenter is not in the PATH but it is in the same folder
          NSENTER=$(dirname "$0")/nsenter
      else
          NSENTER=nsenter
      fi

      if [ -z "$1" ]; then
          echo "Usage: `basename "$0"` CONTAINER [COMMAND [ARG]...]"
          echo ""
          echo "Enters the Docker CONTAINER and executes the specified COMMAND."
          echo "If COMMAND is not specified, runs an interactive shell in CONTAINER."
      else
          PID=$(docker inspect --format "&#123;&#123;.State.Pid}}" "$1")
          [ -z "$PID" ] && exit 1
          shift

          if [ "$(id -u)" -ne "0" ]; then
              which sudo > /dev/null
              if [ "$?" -eq "0" ]; then
                LAZY_SUDO="sudo "
              else
                echo "Warning: Cannot find sudo; Invoking nsenter as the user $USER." >&2
              fi
          fi
          
          # Get environment variables from the container's root process

          ENV=$($LAZY_SUDO cat /proc/$PID/environ | xargs -0)

          # Prepare nsenter flags
          OPTS="--target $PID --mount --uts --ipc --net --pid --"

          # env is to clear all host environment variables and set then anew
          if [ $# -lt 1 ]; then
              # No arguments, default to `su` which executes the default login shell
              $LAZY_SUDO "$NSENTER" $OPTS env -i - $ENV su -m root
          else
              # Has command
              # "$@" is magic in bash, and needs to be in the invocation
              $LAZY_SUDO "$NSENTER" $OPTS env -i - $ENV "$@"
          fi
      fi
  - path: /etc/iptables.rules
    permissions: 0600
    content: |
      *filter
      :INPUT DROP [0:0]
      :FORWARD ACCEPT [0:0]
      :OUTPUT ACCEPT [0:0]
      -A INPUT -i lo -j ACCEPT
      -A INPUT -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
      -A INPUT -i eth0 -p tcp -m conntrack --ctstate NEW -m multiport --dports 22 -j ACCEPT
      -A INPUT -m limit --limit 5/min -j LOG --log-prefix "iptables denied: " --log-level 7
      -A INPUT -j DROP
      COMMIT
      # end of file
ssh_authorized_keys:
  - ssh-rsa AAA...
```

I install a CoreOS to disk with cloud-config.yml file.

``` bash
$ sudo coreos-install -d /dev/vda -C stable -c ~/cloud-config.yml
$ sudo reboot
```

### fleet unit file

I revise a changed ngrok url in fleet unit file of MoinMoin service.

``` bash ~/docker_apps/moin/moin@.service
[Unit]
Description=MoinMoin Service
After=docker.service
[Service]
TimeoutStartSec=0
ExecStartPre=-/usr/bin/docker kill %p%i
ExecStartPre=-/usr/bin/docker rm %p%i
ExecStartPre=/bin/bash -c '/usr/bin/docker start || /usr/bin/docker run -v /usr/local/share/moin/data/pages --name moin-vol xxx.ngrok.com/moin-data true && true'
ExecStartPre=/usr/bin/docker pull xxx.ngrok.com/moin
ExecStart=/usr/bin/docker run --name %p%i --volumes-from moin-vol -p 80:80 xxx.ngrok.com/moin
ExecStop=/usr/bin/docker stop %p%i
[Install]
WantedBy=multi-user.target
[X-Fleet]
X-Conflicts=%p@*.service
MachineMetadata=role=moin
```

Then I run ngrok container to tunnel my private docker-registry.

``` bash
$ docker run -it --rm wizardapps/ngrok:latest ngrok 10.1.1.32:5000
```

And I re-submit the fleet unit file before starting MoinMoin service.
 
``` bash
$ cd ~/docker_apps/moin
$ fleetctl destroy moin@.service
Destroyed moin@.service
$ fleetctl submit moin@.service
$ fleetctl load moin@80.service
Unit moin@80.service loaded on 6ad10563.../108.61.162.139
$ fleetctl start moin@80.service
Unit moin@80.service launched on 6ad10563.../108.61.162.139
```

It takes about 6 minutes for downloading images from docker-registry and finish running process.

```
$ fleetctl journal -f moin@80.service
-- Logs begin at Sun 2014-10-26 08:06:34 UTC. --
...
Oct 26 08:16:45 localhost systemd[1]: Started MoinMoin Service.
```
