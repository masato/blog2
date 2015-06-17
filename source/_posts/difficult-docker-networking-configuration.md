title: "Difficult Docker networking configuration"
date: 2014-11-04 22:39:46
tags:
 - HTTPRouting
 - Nginx
 - nginx-proxy
 - hipache
 - Vultr
 - CoreOS
---

I'm often stuck on Docker networking configuration. I hear about SDN for Docker recently and very interested in networking. I should be be familiar with networking and distributed storage technologies even though I am a programmer.

<!-- more -->

### CoreOS on VPS

I've been playig with Core OS on Vultr in these days, today I realized that a iptables rule which I set on CoreOS is not working in the way it's supposed to do. I didn't know why INPUT rule is now working. The reason of this storange behavior (at least for me) is Docker set DNAT rules by default.

I manage to add a -s flag on an auto generated DNAT rule, and it looks like I intend.

``` bash
$ sudo sh -c "iptables-save > /etc/iptables.rules"
$ sudo vi /etc/iptables.rules
-A DOCKER -s 210.xxx.xxx.xxx/32 ! -i docker0 -p tcp -m tcp --dport 443 -j DNAT --to-destination 172.17.0.3:443
$ sudo iptables-restore /etc/iptables.rules
```

Of course this is a transient work around. When docker restarted this modified pars will disappear. A better solution which comes to mind is combined with automated reverse proxies such a nginx-proxy or hipache.

### References

* [Network Configuration](https://docs.docker.com/articles/networking/)
* [Four ways to connect a docker container to a local network](http://blog.oddbit.com/2014/08/11/four-ways-to-connect-a-docker/)
* [Automated Nginx Reverse Proxy for Docker](http://jasonwilder.com/blog/2014/03/25/automated-nginx-reverse-proxy-for-docker/)
* [Using nginx, confd, and docker for zero-downtime web updates](http://brianketelsen.com/2014/02/25/using-nginx-confd-and-docker-for-zero-downtime-web-updates/)