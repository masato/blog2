title: "DockerのHTTP Routing - Part7: Nginx multiple upstreams"
date: 2014-07-30 20:37:17
tags:
 - HTTPRouting
 - Nginx
 - nginx-proxy
 - confd
 - docker-gen
 - Go
description: xip.ioとNode.jsのマルチプルポートのサンプルで試した内容を、リバースプロキシとして使うNginxで実現するためにメモ
---


* `Update 2014-08-05`: [DockerのHTTP Routing - Part8: xip.io と Nginx と confd 0.6 で動的リバースプロキシ](/2014/08/05/docker-reverse-proxy-xipio-nginx-confd-sinatra/)

[xip.ioとNode.jsのマルチプルポートのサンプル](/2014/07/24/docker-reverse-proxy-bouncy/)で試した内容を、リバースプロキシとして使うNginxで実現するためにメモ。

異なる`server_name`で複数の[serverディレクティブ](http://nginx.org/ja/docs/http/server_names.html)と[upstreamディレクティブ](http://wiki.nginx.org/HttpUpstreamModuleJa)を定義できます。

[docker-gen](https://github.com/jwilder/docker-gen)は[Docker Remote API](http://docs.docker.io/reference/api/docker_remote_api/)のinspectを`/var/run/docker.sock`経由で取得し、[confd](https://github.com/kelseyhightower/confd)はetcdに保存した情報を使い、どちらもGoの[text/template](http://golang.org/pkg/text/template/)を使い、`/etc/nginx/nginx.conf`を生成します。


## upstreamの作り方

* [Automated Nginx Reverse Proxy for Docker](http://jasonwilder.com/blog/2014/03/25/automated-nginx-reverse-proxy-for-docker/)
* [Experimenting with CoreOS, confd, etcd, fleet, and CloudFormation](http://marceldegraaf.net/2014/04/24/experimenting-with-coreos-confd-etcd-fleet-and-cloudformation.html)

