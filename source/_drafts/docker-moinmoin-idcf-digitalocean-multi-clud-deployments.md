title: "MoinMoin in Production on CoreOS - Part8: Multi-cloud deployments IDCF Cloud and DigitalOcean"
date: 2014-10-23 01:01:40
tags:
 - MoinMoin
 - CoreOS
 - IDCFCloud
 - DigitalOcean
---

I am used to run instances behind a virtual router on CloudStack. When I first run droplet on DigitalOcean I am suprised that a instance is assined public ip address and opened to internet without iptables enabled nor behind firewall. So I create a cloud-config.yml with iptables.service.  