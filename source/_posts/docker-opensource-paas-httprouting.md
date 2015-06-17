title: "DockerでオープンソースPaaS - Part2: HTTP Routing"
date: 2014-07-03 23:22:22
tags:
 - Docker
 - DockerPaaS
 - Heroku
 - Hipache
 - Vulcand
 - Namecheap
 - tsuru
 - HTTPRouting
 - ServiceDiscovery
description: DockerでPaaSを構築する場合、課題となるのはDockerのネットワーク周りです。Dockerコンテナはephemeralなので、Dockerホストの外部からアクセスするときのルーティングの仕組みが必要になります。Herokuの場合HTTP Routingがあります。オープンソースでDynamic HTTP Routingを実現するために、RedisをバックエンドにしているHipachを使うと良さそうです。DokkuやtsuruでもHipachでHTTP Routingを実現しています。Build your own platform as a service with Dockerを読んでいると、ワイルドカードDNSの名前解決にNamecheapを使っています。
---

* `Update 2014-08-05`: [DockerのHTTP Routing - Part8: xip.io と Nginx と confd 0.6 で動的リバースプロキシ](/2014/08/05/docker-reverse-proxy-xipio-nginx-confd-sinatra/)
* `Update 2014-08-06`: [DockerのHTTP Routing - Part9: xip.io と OpenResty で動的リバースプロキシ](/2014/08/06/docker-reverse-proxy-xipio-openresty-sinatra/)
* `Update 2014-08-07`: [DockerのHTTP Routing - Part10: xip.io と Hipache で動的リバースプロキシ](/2014/08/07/docker-reverse-proxy-xipio-hipache)

DockerでPaaSを構築する場合、課題となるのはDockerのネットワーク周りです。
Dockerコンテナはephemeralなので、Dockerホストの外部からアクセスするときのルーティングの仕組みが必要になります。Herokuの場合[HTTP Routing](https://devcenter.heroku.com/articles/http-routing)があります。

オープンソースで`Dynamic HTTP Routing`を実現するために、Redisをバックエンドにしている[Hipache](https://github.com/dotcloud/hipache)を使うと良さそうです。DokkuやtsuruでもHipachで`HTTP Routing`を実現しています。

[Build your own platform as a service with Docker](http://serverascode.com/2014/06/16/build-your-own-paas-docker.html)を読んでいると、ワイルドカードDNSの名前解決に[Namecheap](https://www.namecheap.com/)を使っています。

<!-- more -->
 
### Namecheapを契約する

[Namecheap](https://www.namecheap.com/)でドメインを購入します。
パラオの.pwドメインが$3.88/yearでセール中だったので購入しました。
us居住だと.usドメインが$0.98/yearで購入できます。

適当なドメインを購入後に設定をします。

```
My Account > Manage Domains > 購入したドメイン > Modify Domain > All Host Records
```

ここでtsuruをインストールしたホストのIPアドレスと、URLリダイレクトの確認用の設定をします。

| HOST NAME | IP ADDRESS/URL |RECORD TYPE | MX PREF | TTL |
| :-: | :-: | :-: | :-: | :-: |
| @ | {tsusuのホストIPアドレス} | A (Address) | n/a | 60 |
| www | {リダイレクト先のURL} | URL Redirect (301) | n/a | 60 |
| * | {tsusuのホストIPアドレス} | A (Address) | n/a | 60 |

### HipacheとVulcand

tsuruの場合、`HTTP Routing`はRedisをバックエンドに持つ[Hipache](https://github.com/dotcloud/hipache)を使っています。
リバースプロキシの仕組みはまだよくわかっていないのですが、HAProxy+Serfよりも良さそうな気がします。

ワイルドカードでDNSの名前解決ができれば、Hipachaがリバースプロキシとして機能するので、tsuruで作成したアプリへ動的にルーティングしてくれます。ルーティングの情報はRedisに保存されているので、`redis-cli`で確認することができます。

Etcdをバックエンドに持つ[Vulcand](https://github.com/mailgun/vulcand)というのもあり、RedisもEtcdも好きなので、こちらもあとで使ってみようと思います。

### まとめ

.pwドメインは、[お名前.com](https://www.onamae.com)で購入すると680円でした。
Namecheapでは[PositiveSSL Wildcard](https://www.namecheap.com/security/ssl-certificates/wildcard.aspx)も$94/yrで購入できます。



