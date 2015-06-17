title: "DockerでDebian 7 wheezy"
date: 2014-06-26 21:58:27
tags:
 - Docker
 - Debian
 - Tutum
 - InfiniDB
description: GCEのオフィシャルイメージにUbuntuがなくDebianだったりするので、より汎用的なDockerイメージも作っておきたくて、Debianイメージも用意したいと思います。GNU/Linux! InfiniDBのテスト環境を用意するのが目的です。
---

GCEのオフィシャルイメージにUbuntuがなくDebianだったりするので、より汎用的なDockerイメージも作っておきたくて、
Debianイメージも用意したいと思います。`GNU/Linux!`

InfiniDBのテスト環境を用意するのが目的です。



<!-- more -->

### GCEのDebian

あまり関係ないですが、`gcutil listimages`を見ておきます。頻繁にイメージは更新されていますが、wheezyです。

``` bash
$ gcutil listimages
+-------------------------------------------------------------------------+-------------+--------+
| name                                                                    | deprecation | status |
+-------------------------------------------------------------------------+-------------+--------+
| coreos-v310-1-0                                                         |             | READY  |
+-------------------------------------------------------------------------+-------------+--------+
| projects/centos-cloud/global/images/centos-6-v20140619                  |             | READY  |
+-------------------------------------------------------------------------+-------------+--------+
| projects/debian-cloud/global/images/backports-debian-7-wheezy-v20140619 |             | READY  |
+-------------------------------------------------------------------------+-------------+--------+
| projects/debian-cloud/global/images/debian-7-wheezy-v20140619           |             | READY  |
+-------------------------------------------------------------------------+-------------+--------+
| projects/opensuse-cloud/global/images/opensuse-13-1-v20140609           |             | READY  |
+-------------------------------------------------------------------------+-------------+--------+
| projects/opensuse-cloud/global/images/opensuse131-v20140417             | DEPRECATED  | READY  |
+-------------------------------------------------------------------------+-------------+--------+
| projects/rhel-cloud/global/images/rhel-6-v20140619                      |             | READY  |
+-------------------------------------------------------------------------+-------------+--------+
| projects/suse-cloud/global/images/sles-11-sp3-v20140609                 |             | READY  |
+-------------------------------------------------------------------------+-------------+--------+
```

### Docker Hub Registry

自分でDockerfileを書く前に、[Docker Hub Registry](https://registry.hub.docker.com/search?q=debian&s=downloads)から、Download数の多いイメージを検索します。
`google/debian`というのもありますが、SSH接続ができる[tutum/debian](https://registry.hub.docker.com/u/tutum/debian/)を使ってみます。

Tutumのイメージは比較的よく更新されていて、WordPressやLAMPのイメージは有名です。

### イメージの起動

まずは`docker pull`します。

``` bash
$ docker pull tutum/debian:wheezy
```

SSHを2222にポートフォワードしてrunします。

``` bash
$ docker run -d -p 2222:22 -e ROOT_PASS="mypass" tutum/debian:wheezy
```

SSHでログインします。

```
$ ssh root@localhost -p 2222
Warning: Permanently added '[localhost]:2222' (ECDSA) to the list of known hosts.
root@localhost's password:
Linux viper 3.12.20-gentoo #1 SMP Sun May 18 12:36:24 MDT 2014 x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
root@d043021ec661:~#
```

### まとめ

SSHでログインできるDebianのコンテナが簡単にできました。
Dibianはあまり使わないのですが、徐々にUbuntuから移行していきたいです。

DebianコンテナはInfiniDBをインストールするために用意したので、次回はInfiniDBを使ってみます。
