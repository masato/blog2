title: "MoinMoin in Production on CoreOS - Part8: iptables on DigitalOcean"
date: 2014-10-24 02:23:11
tags:
 - CoreOS
 - IDCFCloud
 - DigitalOcean
 - iptables
 - cloud-config
 - fleet
description: DigitalOcean now officially supports CoreOS and cloud-config is integrated. It offers better user experience for providing cloud-config setting via the web console or api than other VPS providers. I often use IDCF Cloud which offers virtual router and each instances resides behind a firewall. When I use CoreOS on DigitalOcean I am a bit surprised that it has no iptables rules and open to the world as default.
---

DigitalOcean now officially supports CoreOS and cloud-config is integrated. It offers better user experience for providing cloud-config setting via the web console or api than other VPS providers. I often use IDCF Cloud which offers virtual router and each instances resides behind a firewall. When I use CoreOS on DigitalOcean I am a bit surprised that it has no iptables rules and open to the world as default.

<!-- more -->

### cloud-config with iptables

Thanks to this [post](http://blog.mapstrata.com/coreos-install-to-a-vps/), I have learned to configure iptables rules. I edited cloud-config data providing for DigitalOcean. 
Do not forget to end with comment line when writing iptables rules in `write_files` directive, it should be end line in rules.

``` yml
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
        Type=oneshot
        ExecStart=/usr/bin/timedatectl set-timezone Asia/Tokyo
        RemainAfterExit=yes
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
      :FORWARD DROP [0:0]
      :OUTPUT ACCEPT [0:0]
      :RH-Firewall-1-INPUT - [0:0]
      -A INPUT -j RH-Firewall-1-INPUT
      -A FORWARD -j RH-Firewall-1-INPUT
      -A RH-Firewall-1-INPUT -i lo -j ACCEPT
      -A RH-Firewall-1-INPUT -p icmp --icmp-type echo-reply -j ACCEPT
      -A RH-Firewall-1-INPUT -p icmp --icmp-type destination-unreachable -j ACCEPT
      -A RH-Firewall-1-INPUT -p icmp --icmp-type time-exceeded -j ACCEPT
      # Block Spoofing IP Addresses
      -A INPUT -i eth0 -s 10.0.0.0/8 -j DROP
      -A INPUT -i eth0 -s 172.16.0.0/12 -j DROP
      -A INPUT -i eth0 -s 192.168.0.0/16 -j DROP
      -A INPUT -i eth0 -s 224.0.0.0/4 -j DROP
      -A INPUT -i eth0 -s 240.0.0.0/5 -j DROP
      -A INPUT -i eth0 -d 127.0.0.0/8 -j DROP
      # Accept Pings
      -A RH-Firewall-1-INPUT -p icmp --icmp-type echo-request -j ACCEPT
      # Accept any established connections
      -A RH-Firewall-1-INPUT -m conntrack --ctstate  ESTABLISHED,RELATED -j ACCEPT
      # Accept ssh, http, https - add other tcp traffic ports here
      -A RH-Firewall-1-INPUT -m conntrack --ctstate NEW -m multiport -p tcp --dports 22,80,443 -j ACCEPT
      #Log and drop everything else
      -A RH-Firewall-1-INPUT -j LOG
      -A RH-Firewall-1-INPUT -j REJECT --reject-with icmp-host-prohibited
      COMMIT
      # end of file
```