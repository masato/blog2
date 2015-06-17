title: "DockerのHTTP Routing: Part2: リバースプロキシ調査"
date: 2014-07-18 23:18:22
tags:
 - Docker
 - HTTPRouting
 - ServiceDiscovery
 - OpenResty
 - bouncy
 - Hipache
 - Lua
 - DockerRemoteAPI
description: Part1に続いて、カジュアルに動的HTTP RotingをするためのリバースプロキシについてStackOverflowとServerFaultで調べてみます。Simple HTTP proxy / routing for my docker containers? Access to a docker instance - Best Practice. port redirect to docker containers by hostname.OpenResty、Hipache、bouncy辺りが良さそうです。
---

* `Update 2014-07-24`: [DockerのHTTP Routing - Part5: xip.io と bouncy と XRegExp で動的リバースプロキシ](/2014/07/24/docker-reverse-proxy-bouncy/)
* `Update 2014-08-05`: [DockerのHTTP Routing - Part8: xip.io と Nginx と confd 0.6 で動的リバースプロキシ](/2014/08/05/docker-reverse-proxy-xipio-nginx-confd-sinatra/)
* `Update 2014-08-06`: [DockerのHTTP Routing - Part9: xip.io と OpenResty で動的リバースプロキシ](/2014/08/06/docker-reverse-proxy-xipio-openresty-sinatra/)
* `Update 2014-08-07`: [DockerのHTTP Routing - Part10: xip.io と Hipache で動的リバースプロキシ](/2014/08/07/docker-reverse-proxy-xipio-hipache-sinatra/)
* `Update 2014-08-09`: [Dockerでデータ分析環境 - Part6: Ubuntu14.04のDockerにRStudio Serverと」OpenRestyをデプロイ](/2014/08/09/docker-analytic-sandbox-rstudio-openresty-deploy/)

[Part1](/2014/07/14/docker-service-discovery-http-routing/)に続いて、カジュアルに動的`HTTP Roting`をするための、リバースプロキシについてStackOverflowとServerFaultで調べてみます。

* [Simple HTTP proxy / routing for my docker containers?](http://stackoverflow.com/questions/21315710/simple-http-proxy-routing-for-my-docker-containers)
* [Access to a docker instance - Best Practice](http://serverfault.com/questions/580321/access-to-a-docker-instance-best-practice)
* [port redirect to docker containers by hostname](http://stackoverflow.com/questions/21570287/port-redirect-to-docker-containers-by-hostname)

[OpenResty](http://openresty.org/), [Hipache](https://github.com/dotcloud/hipache), [bouncy](https://github.com/substack/bouncy)辺りが良さそうです。

### OpenRestyとLua

OpenRestyは、Luaのパッケージ管理ツールの[LuaRocks](http://luarocks.org/)も同梱されているので、Luaのプログラミングがすぐできる状態になっています。
OpenRestyはLuaの勉強環境としても便利に使えそうなのでいろいろ試してみたいです。

* LuaやmrubyでNginxの動的処理を書ける
* Cocos2d-xのロジック実装言語に使える
* GeventみたいなCoroutineが書ける

[Gin](http://gin.io/)を使うとOpenRestyとLuaだけでJSONを返す`REST API`サーバーが作れます。
SPAのバックエンドにGinというのもおもしろそう。

### 3scale/openresty

Dockerイメージは[3scale/openresty](https://registry.hub.docker.com/u/3scale/openresty/)を使ってみようと思います。
[3scale](http://www.3scale.net/)は、Mashery、Apigee、Layer7と同じカテゴリでAPI管理サービスを提供しています。

Ginの例もあって、なるほどOpenRestyとAPI管理サーバーは相性が良さそうです。

[今、起こりつつあるAPIエコノミーとか何か？](http://jp.techcrunch.com/2013/04/29/20130428facebook-and-the-sudden-wake-up-about-the-api-economy/)を再読してAPI管理について、特に社内システムのAPIを考えたいです。
GitHubは[docker-openresty](https://github.com/3scale/docker-openresty)です。

実際にを触ってみながら、使いやすいツールを決めようと思います。












