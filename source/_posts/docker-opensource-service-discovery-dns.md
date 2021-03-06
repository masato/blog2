title: "DockerでオープンソースPaaS - Part6: DNS Forwrding"
date: 2014-07-07 23:33:10
tags:
 - DockerPaaS
 - tsuru
 - ServiceDiscovery
 - SkyDNS
 - DNSForwarding
 - IDCFクラウド
description: まだtsuruでアプリをデプロイが失敗中です。いわゆるService Discoveryですが、Wildcard DNSとHipacheだけでは足りなくて、ローカルにDNSサーバーを建てる必要があるみたいです。tsuruでドメイン名を指定するとLAN内でもグローバルIPアドレスで他のコンテナを探してしまい見つかりません。dnsmasqとhostsで対応しようと思ったのですが、Wildcardが使えないのでした。
---

まだtsuruでアプリをデプロイが失敗中です。いわゆる`Service Discovery`ですが、
`Wildcard DNS`とHipacheだけでは足りなくて、ローカルにDNSサーバーを建てる必要があるみたいです。
tsuruでドメイン名を指定するとLAN内でもグローバルIPアドレスで他のコンテナを探してしまい見つかりません。
dnsmasqとhostsで対応しようと思ったのですが、Wildcardが使えないのでした。

<!-- more -->

### DNS Forwarding

`DNS Forwarding`用のBindの構築方法は、DigitalOceanの[How To Configure Bind as a Caching or Forwarding DNS Server on Ubuntu 14.04](https://www.digitalocean.com/community/tutorials/how-to-configure-bind-as-a-caching-or-forwarding-dns-server-on-ubuntu-14-04)が詳しいので参考にします。

tsuruにも、[Howto install a dns forwarder](http://docs.tsuru.io/en/latest/misc/dns-forwarders.html)のドキュメントがあります。

Bindをインストールします。

``` bash
$ sudo apt-get -y install bind9 bind9utils
```

`DNS Forwarders`の設定

named.conf.optionsの設定は、DigitalOceanを参考に書きました。

``` bash /etc/bind/named.conf.options
$ egrep -v '//|^$' /etc/bind/named.conf.options
acl goodclients {
        10.1.1.17;
        localhost;
        localnets;
};
options {
        directory "/var/cache/bind";
        recursion yes;
        allow-query { goodclients; };
        forwarders {
                8.8.8.8;
                8.8.4.4;
        };
        forward only;
        dnssec-enable yes;
        dnssec-validation yes;
        auth-nxdomain no;    # conform to RFC1035
        listen-on-v6 { any; };
};
```

ゾーンファイルは別ファイルに書きます。

``` bash /etc/bind/named.conf.local
zone "masato.pw" {
        type master;
        file "/etc/bind/db.masato.pw";
};
```

ゾーンファイルはtsuruのドキュメントを参考に書きました。

``` bash /etc/bind/db.masato.pw
;
$TTL    604800
@       IN      SOA     masato.pw. tsuru.masato.pw. (
                              3         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL
;
@       IN      NS      masato.pw.
@       IN      A       10.1.1.17
git     IN      A       10.1.1.17 ; here we can set a better exhibition for the git remote provided by tsuru
*       IN      A       10.1.1.17
```

Bindを再起動します。

``` bash
$ sudo named-checkconf
$ sudo service bind9 restart
```

### IDCFクラウドのresolv.conf

IDCFクラウドの場合、DHCP経由でDNSサーバーの情報が配布されます。

``` bash /etc/resolv.conf
nameserver 10.1.0.1
search cs29dcloud.internal
```

DHCPでDNSサーバーの情報を受けつかない方法を調べたのですが、うまく機能しないので、
headに情報を追加して、配布されるDNSサーバーの前に来るようにしました。

``` bash /etc/resolvconf/resolv.conf.d/head
nameserver 10.1.1.17
search masato.pw
```

resolv.confを更新します。

``` bash
$ sudo resolvconf -u
```

うまく設定が入りました。

``` bash /etc/resolv.conf
nameserver 10.1.1.17
search masato.pw


nameserver 10.1.0.1
search cs29dcloud.internal
```

Dockerホストのresolv.confを変更した場合、Dockerコンテナのresolv.confが変更できなかったので、
一度破棄してrunし直します。

### Dockerホストでping確認

`Wildcard DNS`がプライベートIPアドレスを返すようになったので、ローカルDNSで名前解決されるようになりました。

``` bash
$ ping -c 2 blog.masato.pw
PING blog.masato.pw (10.1.1.17) 56(84) bytes of data.
64 bytes from 10.1.1.17: icmp_seq=1 ttl=64 time=0.023 ms
64 bytes from 10.1.1.17: icmp_seq=2 ttl=64 time=0.038 ms
```

### tusuアプリ(Dockerコンテナ)でping確認

tsuruアプリにSSH接続します。

``` bash
$ ssh ubuntu@172.17.0.19 -i /var/lib/tsuru/.ssh/id_rsa
Welcome to Ubuntu 14.04 LTS (GNU/Linux 3.13.0-24-generic x86_64)
```

resolv.confもローカルのDNSサーバーが先に書かれています。

``` bash
$ cat /etc/resolv.conf
# Dynamic resolv.conf(5) file for glibc resolver(3) generated by resolvconf(8)
#     DO NOT EDIT THIS FILE BY HAND -- YOUR CHANGES WILL BE OVERWRITTEN
nameserver 10.1.1.17
search masato.pw


nameserver 10.1.0.1
search cs29dcloud.internal
```

pingすると、ローカルのDNSで名前解決ができています。

``` bash
$ sudo ping -c 2 tsuru-dashboard.masato.pw
PING tsuru-dashboard.masato.pw (10.1.1.17) 56(84) bytes of data.
64 bytes from 10.1.1.17: icmp_seq=1 ttl=64 time=0.025 ms
64 bytes from 10.1.1.17: icmp_seq=2 ttl=64 time=0.051 ms
```

### まとめ

ローカルDNSで名前解決ができるようになったので、[トラブルシュートの手順](/2014/07/05/docker-opensource-paas-tsuru-install-troubleshoot/)で、tsuruを再インストールします。

ようやくtsuruアプリをデプロイできる環境になった気がします。

