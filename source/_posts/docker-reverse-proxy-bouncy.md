title: "DockerのHTTP Routing - Part5: xip.io と bouncy と XRegExp で動的リバースプロキシ"
date: 2014-07-24 22:01:22
tags:
 - HTTPRouting
 - bouncy
 - xipio
 - XRegExp
 - Nodejs
description: 前回NginxでリバースプロキシをしたマルチポートのNode.jsサンプルをbouncyで試してみます。Dockerコンテナのリバースプロキシには使えませんが、簡単なNode.jsアプリの場合、カジュアルにリバースプロキシできます。Nginxのserver_nameの正規表現の名前付きキャプチャは、Node.jsのXRegExpを使います。fluentdではRubyの正規表現で名前付きキャプチャをよく利用しますが、Node.jsの場合はXRegExpで同じことができます。
---

* `Update 2014-08-05`: [DockerのHTTP Routing - Part8: xip.io と Nginx と confd 0.6 で動的リバースプロキシ](/2014/08/05/docker-reverse-proxy-xipio-nginx-confd-sinatra/)
* `Update 2014-08-06`: [DockerのHTTP Routing - Part9: xip.io と OpenResty で動的リバースプロキシ](/2014/08/06/docker-reverse-proxy-xipio-openresty-sinatra/)
* `Update 2014-08-07`: [DockerのHTTP Routing - Part10: xip.io と Hipache で動的リバースプロキシ](/2014/08/07/docker-reverse-proxy-xipio-hipache-sinatra/)

前回Nginxでリバースプロキシをした[マルチポートのNode.js](/2014/07/23/docker-reverse-proxy-xipio-nginx/)サンプルを[bouncy](https://github.com/substack/bouncy)で試してみます。Dockerコンテナのリバースプロキシには使えませんが、簡単なNode.jsアプリの場合、カジュアルにリバースプロキシできます。

Nginxの`server_name`の正規表現の名前付きキャプチャは、Node.jsの[XRegExp](https://github.com/slevithan/xregexp)を使います。
fluentdではRubyの正規表現で名前付きキャプチャをよく利用しますが、Node.jsの場合はXRegExpで同じことができます。

<!-- more -->

### Dockerfile

Dockerfileは[前回](/2014/07/23/docker-reverse-proxy-xipio-nginx/)とほぼ同じで、Node.jsアプリのデプロイと、runitの設定ファイルだけ変更します。

``` bash ~/docker_apps/bouncy/Dockerfile

FROM phusion/baseimage:0.9.11

# Set correct environment variables.
ENV HOME /root

# Use baseimage-docker's init process.
CMD ["/sbin/my_init"]

# ...put your own build instructions here...

# apt-get update
RUN sed -i~ -e 's/archive.ubuntu.com/ftp.jaist.ac.jp/' /etc/apt/sources.list \
 && apt-get update

# ppa
RUN apt-get install -y software-properties-common
RUN add-apt-repository ppa:chris-lea/node.js
RUN apt-get -yqq update

# apt-get install
RUN apt-get install -y nodejs

# deploy Node.js app
RUN rm -rf /etc/nginx/sites-enabled/default
ADD sv /etc/service
ADD webapp /srv/www/webapp/
RUN cd /srv/www/webapp/ && npm install

# Clean up APT when done.
RUN apt-get clean && rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*
```

### Node.jsアプリ

package.jsonでデプロイするNode.jsアプリで利用するパッケージを定義します。

``` node ~/docker_apps/bouncy/webapp/package.json
{
  "name": "bouncy-proxy",
  "description": "Minimal bouncy proxy",
  "version": "0.0.1",
  "dependencies": {
    "bouncy": "*"
   ,"xregexp": "*"
  }
}
```

[bouncy](https://github.com/substack/bouncy)は`bounce HTTP requests around for load balancing or as an HTTP host router`とREADMEに書かれているように、Nginxの代換えになります。

使い方はとても簡単です。Node.jsのアプリとして80で起動します。
[XRegExp](https://github.com/slevithan/xregexp)でhost名からポート名を名前付きキャプチャします。


``` node  ~/docker_apps/bouncy/proxy.js
var bouncy = require('bouncy')
  , XRegExp = require('xregexp').XRegExp;

var server = bouncy(function (req, res, bounce) {
  var port = XRegExp('^p(?<port_name>\\d+)\\.(.*)\\.xip\\.io$');
  var match = XRegExp.exec(req.headers.host, port);
  if (match) {
    bounce(match.port_name);
  }
  else {
    res.statusCode = 404;
    res.end('no such host');
  }
});
server.listen(80);
```

### runitの設定ファイル

bouncyのrunitの設定ファイルです。

``` bash ~/docker_apps/bouncy/sv/bouncy/run
#!/bin/sh

cd /srv/www/webapp
exec 2>&1
exec chpst -U www-data node proxy.js > /var/log/bouncy.log 2>&1
```

### 確認

Dockerイメージをbuildします。

```
$ docker build -t masato/bouncy .
```

80ポートをポートマップしてDockerコンテナを起動します。

``` bash
$ docker run --rm -i -p 80:80 -t masato/bouncy /sbin/my_init bash
```

DockerコンテナのIPアドレスを確認します。

``` bash
$ docker inspect 26aa2ff50a74 | jq -r '.[0].NetworkSettings.IPAddress'
172.17.1.22
```

Dockerホストから、xip.ioを使ったURLを指定してブラウザで確認します。

{% img center /2014/07/24/docker-reverse-proxy-bouncy/bouncy.png %}


### まとめ

Dockerコンテナのリバースプロキシ用途ではありませんが、Node.jsアプリのリバースプロキシにNginxをインストールしなくても、Node.jsだけでカジュアルに実現できます。

また、XRegExpはJavaScriptの正規表現を拡張して、名前付きキャプチャ以外にも多くの機能が実装されています。
ブラウザでもNode.jsでもインストールできるので、フォームのバリデーションで使えそうです。


