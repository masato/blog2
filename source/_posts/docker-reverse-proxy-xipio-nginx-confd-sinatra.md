title: "DockerのHTTP Routing - Part8: xip.io と Nginx と confd 0.6 で動的リバースプロキシ"
date: 2014-08-05 00:37:39
tags:
 - まとめ
 - Docker
 - confd
 - HTTPRouting
 - Nginx
 - nginx-proxy
 - xipio
 - Sinatra
 - DataVolumeContainer
description: 以下のサイトを参考に、confdとNginxで動的リバースプロキシを作成してみます。特にconfdは新しいテンプレートを使いたいので、0.6.0-alpha1を利用しています。Experimenting with CoreOS, confd, etcd, fleet, and CloudFormation, Automated Nginx Reverse Proxy for Docker, confd 0.6.x, xip.ioのサブドメインにリバースプロキシ先のポート番号をprefixして、Nginxは80ポートで複数のserver_nameに応答することができます。
---

* `Update 2014-08-06`: [DockerのHTTP Routing - Part9: xip.io と OpenResty で動的リバースプロキシ](/2014/08/06/docker-reverse-proxy-xipio-openresty-sinatra/)
* `Update 2014-08-07`: [DockerのHTTP Routing - Part10: xip.io と Hipache で動的リバースプロキシ](/2014/08/07/docker-reverse-proxy-xipio-hipache-sinatra/)

以下のサイトを参考に、confdとNginxで動的リバースプロキシを作成してみます。
特にconfdは新しいテンプレートを使いたいので、`0.6.0-alpha1`を利用しています。

* [Experimenting with CoreOS, confd, etcd, fleet, and CloudFormation](http://marceldegraaf.net/2014/04/24/experimenting-with-coreos-confd-etcd-fleet-and-cloudformation.html)
* [Automated Nginx Reverse Proxy for Docker](http://jasonwilder.com/blog/2014/03/25/automated-nginx-reverse-proxy-for-docker/)
* [confd 0.6.x](https://github.com/kelseyhightower/confd/blob/0.6.x/docs/quick-start-guide.md)

xip.ioのサブドメインにリバースプロキシ先のポート番号をprefixして、Nginxは80ポートで複数の`server_name`に応答することができます。

<!-- more -->

### データコンテナの作成

Dockerfileを作成します。VOLUMEに格納するディレクトリを書きます。nsenterでアタッチするときにbashが必要になるので、busyboxではなくubuntuにしました。

``` bash ~/docker_apps/confd-data/Dockerfile
FROM ubuntu:trusty
MAINTAINER Masato Shimizu <ma6ato@gmail.com>

VOLUME ["/etc/confd","/registry_conf","/data","/etc/supervisor/conf.d","/etc/nginx/sites-enabled"]
CMD ["/bin/bash"]
```

イメージをビルドします。

``` bash
$ docker build -t masato/confd-data .
```

データコンテナを起動します。

``` bash
$ docker run --name confd-data -i -t -d masato/confd-data /bin/bash
```

### condの設定

nsenterを使い、データコンテナにアタッチします。
[前回](/2014/08/03/docker-volume-nsenter-mount/)作成した`~/bin/docker-nester`コマンドを使います。

``` bash
$ docker-nsenter cd887aad372d
```

データコンテナに`/etc/confd/conf.d/nginx.toml`ファイルを作成します。まずディレクトリの作成から。

``` bash
# mkdir -p /etc/confd/conf.d
```

nginx.tomlファイルを作成します。

myapp1とmyapp2のサブドメイン毎に2つ設定します。これは[0.6.x](https://github.com/kelseyhightower/confd/blob/0.6.x/docs/quick-start-guide.md#create-the-source-template-1)から採用された書式です。


``` bash /etc/confd/conf.d/myapp1.nginx.toml
[template]
prefix = "/myapp1"
keys = [
  "/subdomain",
  "/upstream",
]
owner       = "nginx"
mode        = "0644"
src         = "nginx.conf.tmpl"
dest        = "/etc/nginx/sites-enabled/myapp1.conf"
check_cmd   = "/usr/sbin/nginx -t -c /etc/nginx/nginx.conf"
reload_cmd  = "/usr/sbin/service nginx reload"
```

myapp2サブドメイン用のTOMLファイルです。

``` bash /etc/confd/conf.d/myapp2.nginx.toml
[template]
prefix = "/myapp2"
keys = [
  "/subdomain",
  "/upstream",
]
owner       = "nginx"
mode        = "0644"
src         = "nginx.conf.tmpl"
dest        = "/etc/nginx/sites-enabled/myapp2.conf"
check_cmd   = "/usr/sbin/nginx -t -c /etc/nginx/nginx.conf"
reload_cmd  = "/usr/sbin/service nginx reload"
```

[DockerのHTTP Routing - Part7: Nginx multiple upstreams](/2014/07/30/docker-reverse-proxy-nginx-multiple-upstreams/)で調べた書き方で、Nginxの設定ファイルのテンプレートを作成します。`0.6.x`から`getv`と`getvs`が使えます。

Goの[text/template](http://golang.org/pkg/text/template/)を使っていますが、Railsに慣れていると表現力が乏しいです。特にconfdではユーザー定義関数のFuncMapが自分で追加できないので不便です。テンプレート内ではパスを操作したい場合があるので、[strings.Split](http://golang.org/pkg/strings/#Split)も追加して欲しいです。


``` html /etc/confd/templates/nginx.conf.tmpl
upstream &#123;&#123;getv "/subdomain"}} {
&#123;&#123;range getvs "/upstream/*"}}
    server &#123;&#123;.&#125;&#125;;
&#123;&#123;end}}
}

server {
  listen 80;
  server_name ~^&#123;&#123;getv "/subdomain"&#125;&#125;\.(.*)\.xip\.io$;

  location / {
    proxy_set_header host $host;
    proxy_set_header x-real-ip $remote_addr;
    proxy_set_header x-forwarded-for $proxy_add_x_forwarded_for;
    proxy_pass http://&#123;&#123;getv "/subdomain"&#125;&#125;;
  }
}
```

### Nginx + etcd + confdのイメージを作成


今回作成したDockerfileです。etcd, confd, Nginxを1つのコンテナにインストールして使います。

etcdのインストールはいろいろ方法がありますがGOPATHを指定してから`go get`することにします。

confdも`go get`したかったのですがGitHubからタグをしてしてインストールできないようなので、バイナリをダウンロードします。
[0.6.x Quick Start Guide](https://github.com/kelseyhightower/confd/blob/0.6.x/docs/quick-start-guide.md#create-the-source-template-1)で採用された書式を利用します。

``` bash ~/docker_apps/nginx-confd/Dockerfile
FROM dockerfile/supervisor
MAINTAINER Masato Shimizu <ma6ato@gmail.com>
# apt-get update
RUN sed -i~ -e 's/archive.ubuntu.com/ftp.jaist.ac.jp/' /etc/apt/sources.list \
  && apt-get update
# nginx install
RUN DEBIAN_FRONTEND=noninteractive apt-get install -y software-properties-common \
    && add-apt-repository -y ppa:nginx/stable \
    && apt-get update \
    && DEBIAN_FRONTEND=noninteractive apt-get install -y nginx \
    && sed -i 's/^\t\(listen \[::\]:80 .*\)$/\t#\1/' /etc/nginx/sites-available/default
# Go install
RUN DEBIAN_FRONTEND=noninteractive apt-get install -y golang
ENV GOPATH /gopath
ENV PATH $PATH:$GOPATH/bin
# etcd and confd install
RUN go get github.com/coreos/etcd
RUN go get github.com/coreos/etcdctl
RUN  curl -qL https://github.com/kelseyhightower/confd/releases/download/v0.6.0-alpha1/confd-0.6.0-alpha1-linux-amd64 \
     -o /usr/local/bin/confd && chmod +x /usr/local/bin/confd
CMD ["supervisord", "-c", "/etc/supervisor/supervisord.conf"]
EXPOSE 80 4001 7001
```

###  Nginx + etcd + confdンテナの起動

データコンテナを`--volumes-from`に指定して、Nginxのコンテナを起動します。

``` bash
$ docker run --name nginx-confd -d -p 80:80 --volumes-from confd-data masato/nginx-confd
```

起動したコンテナのIPアドレスを確認します。

``` bash
$ docker inspect 095b93ed0e25 | jq -r '.[0].NetworkSettings.IPAddress'
172.17.0.209
```

### etcdの動作確認

Dockerホストからetcdへputとgetのテストをします。

``` bash
$ curl -L http://172.17.0.209:4001/v2/keys/mykey -XPUT -d value="this is awesome"
{"action":"set","node":{"key":"/mykey","value":"this is awesome","modifiedIndex":3,"createdIndex":3}}
$ curl -L http://172.17.0.209:4001/v2/keys/mykey
{"action":"get","node":{"key":"/mykey","value":"this is awesome","modifiedIndex":3,"createdIndex":3}}
```

Dockerホストのetcdctlからも`peers`を指定して確認してみます。etcdクラスタに参加しないので、`-no-sync`オプションを指定します。

``` bash
$ etcdctl -C "172.17.0.209:4001" -no-sync get /mykey
this is awesome
```

### Sinatraコンテナを2つ起動

Sinatraのサンプルアプリは[marceldegraaf/sinatra](https://registry.hub.docker.com/u/marceldegraaf/sinatra/)を利用します。それぞれポートは5001と5002にマッピングします。

``` bash
$ docker pull marceldegraaf/sinatra
$ docker run --name sinatra-5001 -p 5001:5000 -e PORT=5000 -d marceldegraaf/sinatra
$ docker run --name sinatra-5002 -p 5002:5000 -e PORT=5000 -d marceldegraaf/sinatra
```

### Sinatraコンテナをconfdに登録

DockerホストからNginxコンテナのIPアドレスを確認します。

``` bash
$ docker-get-ip 095b93ed0e25
172.17.0.209
```

5001ポートでリバースプロキシするSinatraコンテナのIPアドレスを確認します。

``` bash
$ docker-get-ip b6ea1bb67b7f
172.17.0.37
```

5002ポートでリバースプロキシするSinatraコンテナのIPアドレスを確認します。

``` bash
$ docker-get-ip 93f545c79ce0
172.17.0.191
```

etcdctlコマンドで動的にルーティングを設定します。Dockerホストのetcdはクラスタに参加しないため`-no-sync`を指定します。

`/myapp1/subdomain`のキーに、upstreamの名前を登録します。
`/myapp1/upstream`のキーに、upstream先のserverのIPアドレスとポートを登録します。

``` bash
$ etcdctl -C "172.17.0.209:4001" -no-sync set /myapp1/subdomain p5001
$ etcdctl -C "172.17.0.209:4001" -no-sync set /myapp1/upstream/p5001 172.17.0.37:5000
$ etcdctl -C "172.17.0.209:4001" -no-sync set /myapp2/subdomain p5002
$ etcdctl -C "172.17.0.209:4001" -no-sync set /myapp2/upstream/p5002 172.17.0.191:5000
```

### 確認

xip.ioを使い、NginxからSinatraアプリへ動的リバースプロキシができました。

``` bash
$ curl p5001.10.1.2.164.xip.io
Hello world!
$ curl p5002.10.1.2.164.xip.io
Hello world!
```

confdが作成した、myapp1(p5001サブドメイン)用のNginxの設定ファイルです。

``` bash  /etc/nginx/sites-enabled/myapp1.conf
upstream p5001 {

    server 172.17.0.37:5000;

}

server {
  listen 80;
  server_name ~^p5001\.(.*)\.xip\.io$;

  location / {
    proxy_set_header host $host;
    proxy_set_header x-real-ip $remote_addr;
    proxy_set_header x-forwarded-for $proxy_add_x_forwarded_for;
    proxy_pass http://p5001;
  }
}
```


confdが作成した、myapp2(p5002サブドメイン)用のNginxの設定ファイルです。

``` bash /etc/nginx/sites-enabled/myapp2.conf
upstream p5002 {

    server 172.17.0.191:5000;

}

server {
  listen 80;
  server_name ~^p5002\.(.*)\.xip\.io$;

  location / {
    proxy_set_header host $host;
    proxy_set_header x-real-ip $remote_addr;
    proxy_set_header x-forwarded-for $proxy_add_x_forwarded_for;
    proxy_pass http://p5002;
  }
}
```

### まとめ

* [DockerのHTTP Routing: Part1: Service Discovery](/2014/07/14/docker-service-discovery-http-routing/)
* [DockerのHTTP Routing: Part2: リバースプロキシ調査](/2014/07/18/docker-reverse-proxy/)
* [DockerのHTTP Routing - Part3: OpenResty はじめに](/2014/07/21/docker-reverse-proxy-openresty/)
* [DockerのHTTP Routing - Part4: xip.io と Nginx と Node.js](/2014/07/23/docker-reverse-proxy-xipio-nginx/)
* [DockerのHTTP Routing - Part5: xip.io と bouncy と XRegExp](/2014/07/24/docker-reverse-proxy-bouncy/)
* [DockerのHTTP Routing - Part6: confd はじめに](/2014/07/27/docker-reverse-proxy-nginx-confd/)
* [DockerのHTTP Routing - Part7: Nginx multiple upstreams](/2014/07/30/docker-reverse-proxy-nginx-multiple-upstreams/)
