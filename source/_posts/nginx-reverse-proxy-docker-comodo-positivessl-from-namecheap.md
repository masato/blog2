title: "Nginx as a reverse proxy for Docker - Part1: Comodo PositiveSSL from Namecheap"
date: 2014-11-02 03:20:25
tags:
 - Vultr
 - Namecheap
 - SSL
 - PositiveSSL
 - Nginx
 - nginx-proxy
 - HTTPRouting
 - MoinMoin
description: It is sufficient to use a Self-Signed SSL Certificate in my develop environment on IDCF Cloud. When it comes to the prodoction environment on Vultr, it should be better to purchase a CA-Signed SSL Certificate. Of course cheaper is better for my personal use, I choice a Cheap Comodo PositiveSSL Certificate from Namecheap. I already bought a domain name from Namecheap, it is a convenient way using same registered my account. And what is better my confidential information it protected with WhoisGuard.
---

It is sufficient to use a Self-Signed SSL Certificate in my develop environment on IDCF Cloud. When it comes to the prodoction environment on Vultr, it should be better to purchase a CA-Signed SSL Certificate. Of course cheaper is better for my personal use, I choice a Cheap [Comodo PositiveSSL Certificate](https://www.namecheap.com/security/ssl-certificates/comodo.aspx) from Namecheap. I already bought a domain name from Namecheap, it is a convenient way using same registered my account. And what is better my confidential information it protected with [WhoisGuard](https://www.namecheap.com/security/whoisguard.aspx).

<!-- more -->

### References

To install a SSL Certificate on a Nginx, following these instructions.

* [How to install a PositiveSSL certificate provided by Namecheap](https://kerrygallagher.co.uk/how-to-install-a-positivessl-certificate-provided-by-namecheap/)
* [How do I activate an SSL certificate?](https://www.namecheap.com/support/knowledgebase/article.aspx/794/67/how-to-activate-ssl-certificate)
* [Using Namecheap’s Free SSL with Nginx](http://kbeezie.com/free-ssl-with-nginx/)

### The next refactoring 

Now that my MoinMoin container is replaced with a CA-Signed SSL Certificate, the next refactoring is to separate roles of containers because my MoinMoin container has a taste of monolithic. It's a better micro service way. One is for a reverse proxy and the other is for a uWSGI server.

