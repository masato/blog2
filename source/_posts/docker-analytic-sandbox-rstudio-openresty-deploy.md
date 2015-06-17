title: "Dockerでデータ分析環境 - Part6: Ubuntu14.04のDockerにRStudio ServerとOpenRestyをデプロイ"
date: 2014-08-09 12:31:06
tags:
 - AnalyticSandbox
 - docker-registry
 - nginx-proxy
 - OpenResty
 - RStudioServer
 - xipio
 - IDCFオブジェクトストレージ
 - DataVolumeContainer
 - SSL
 - オレオレ証明書
description: OpenRestyを使ったDocker用のHTTP Proxyができあがったので、ようやく本来の目的であるDokcerを使ったRStudio Serverの学習環境のデプロイをする準備が整いました。開発環境で作成したDockerイメージを、IDCFオブジェクトストレージをバックエンドに設定したdocker-registryにpushして、本番環境のUbuntu14.04にDocker1.1.2からpullして使います。
---

* `Update 2014-09-29`: [Dockerデータ分析環境 - Part8: Gorilla REPLとCMDのcdでno such file or directoryになる場合](/2014/09/29/docker-analytic-sandbox-gorilla-repl/)
* `Update 2014-08-10`: [Dockerでデータ分析環境 - Part7: Ubuntu14.04のDockerにIPython Notebookをデプロイ](/2014/08/10/docker-analytic-sandbox-ipython-notebook-deploy/)

OpenRestyを使ったDocker用の[HTTP Proxy](/2014/08/06/docker-reverse-proxy-xipio-openresty-sinatra/)ができあがったので、
ようやく本来の目的であるDokcerを使った[RStudio Serverの学習環境](/2014/07/09/docker-analytic-sandbox-rstudio-server/)のデプロイをする準備が整いました。

開発環境で作成したDockerイメージを、IDCFオブジェクトストレージをバックエンドに設定したdocker-registryにpushして、本番環境のUbuntu14.04にDocker1.1.2からpullして使います。

<!-- more -->


### docker-registry

本番環境のDockerホストにデプロイするDockerイメージをdocker-registryにpushします。
[docker-registry](http://masato.github.io/2014/05/22/idcf-storage-docker-registry/)のバックエンドはIDCFオブジェクトストレージを使っています。

`RStudio Server`のイメージをpushします。

``` bash
$ docker tag masato/rstudio-server localhost:5000/rstudio-server
$ docker push localhost:5000/rstudio-server
...
Pushing tag for rev [710a7212b5b7] on {http://localhost:5000/v1/repositories/rstudio-data/tags/latest}
```

用意した[Data-onlyコンテナ](/2014/08/08/docker-analytic-sandbox-rstudio-data-only-container/)をpushします。

``` bash
$ docker tag masato/rstudio-data localhost:5000/rstudio-data
$ docker push localhost:5000/rstudio-data
```

用意した[OpenResty](/2014/08/06/docker-reverse-proxy-xipio-openresty-sinatra/)のイメージをpushします。

``` bash
$ docker tag masato/openresty localhost:5000/openresty
$ docker push localhost:5000/openresty
```

### OpenRestyのSSL対応

この前作成したOpenRestyをSSL対応するようにします。

Dockerfileでオレオレ証明書の作成を追加します。

``` bash ~/docker_apps/openresty/Dockerfile
FROM 3scale/openresty
MAINTAINER Masato Shimizu <ma6ato@gmail.com>
RUN useradd --system nginx
ADD nginx.conf /var/www/
ADD openresty.conf /etc/supervisor/conf.d/openresty.conf
RUN sed -i 's/bind 127.0.0.1/bind 0.0.0.0/g' /etc/redis/redis.conf
RUN sed -i~ -e 's/archive.ubuntu.com/ftp.jaist.ac.jp/' /etc/apt/sources.list \
 && apt-get update
RUN DEBIAN_FRONTEND=noninteractive apt-get -y install openssl
RUN cd /etc/ssl \
 && openssl genrsa -out ssl.key 4096 \
 && openssl req -new -batch -key ssl.key -out ssl.csr \
 && openssl x509 -req -days 3650 -in ssl.csr -signkey ssl.key -out ssl.crt
EXPOSE 80 443 6379
```

NginxでSSLのリバースプロキシをする場合、`RStudio Server`のようなセキュリティチェックが厳しいアプリはちょっと嵌まりました。
[nginx: Setup SSL Reverse Proxy (Load Balanced SSL Proxy)](http://www.cyberciti.biz/faq/howto-linux-unix-setup-nginx-ssl-proxy/)を参考にして設定ファイルを作成しました。`proxy_set_header`や`add_header`が必要になります。

80で受けた接続は、443に`return 301`を返してリダイレクトさせます。

``` bash ~/docker_apps/openresty/nginx.conf
worker_processes  1;
error_log  /dev/stderr debug;
daemon off;
events {
    worker_connections  256;
}
http {
    server {
      listen 80;
      return 301 https:/$host$request_uri;
    }
    server {
        listen  443 ssl deferred;
        client_max_body_size 20M;
        ssl on;
        server_name  localhost;
        proxy_buffer_size 64K;
        proxy_buffers 32 48K;
        proxy_busy_buffers_size 256K;
        ### SSL log files ###
        #access_log      /var/log/ssl-access.log;
        #error_log       /var/log/ssl-error.log;
        ### SSL cert files ###
        ssl_certificate /etc/ssl/ssl.crt;
        ssl_certificate_key /etc/ssl/ssl.key;
        ### Add SSL specific settings here ###
        ssl_protocols        SSLv3 TLSv1 TLSv1.1 TLSv1.2;
        ssl_ciphers RC4:HIGH:!aNULL:!MD5;
        ssl_prefer_server_ciphers on;
        keepalive_timeout    60;
        ssl_session_cache    shared:SSL:10m;
        ssl_session_timeout  10m;
        ### force timeouts if one of backend is died ##
        proxy_next_upstream error timeout invalid_header http_500 http_502 http_503 http_504;
        ### Set headers ####
        proxy_set_header        Accept-Encoding   "";
        proxy_set_header        Host            $host;
        proxy_set_header        X-Real-IP       $remote_addr;
        proxy_set_header        X-Forwarded-For $proxy_add_x_forwarded_for;
        ### Most PHP, Python, Rails, Java App can use this header ###
        #proxy_set_header X-Forwarded-Proto https;##
        #This is better##
        proxy_set_header        X-Forwarded-Proto $scheme;
        add_header              Front-End-Https   on;
        ### By default we don't want to redirect it ####
        proxy_redirect     off;
        location / {
            set $upstream "";
            rewrite_by_lua '
               local res = ngx.location.capture("/redis")
               if res.status == ngx.HTTP_OK then
                  ngx.var.upstream       = res.body
               else
                  ngx.exit(ngx.HTTP_FORBIDDEN)
               end
            ';
            proxy_pass http://$upstream;
        }

        location /redis {
             internal;
             set            $redis_key $host;
             redis_pass     127.0.0.1:6379;
             default_type   text/html;
        }
    }
}
```

### Ubuntu14.04のDocker本番環境の作成

Ubuntu14.04にDockerをインストールします。`apt-get install docker.io`でインストールするバージョンは少し古いので、[https://get.docker.io/ubuntu](https://get.docker.io/ubuntu)からインストールします。

``` bash
$ sudo apt-get update
$ sudo apt-get install -y apt-transport-https
$ echo "deb https://get.docker.io/ubuntu docker main" | sudo tee -a  /etc/apt/sources.list.d/docker.list
$ sudo apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv-keys 36A1D7869245C8950F966E92D8576A8BA88D21E9
$ sudo apt-get update
$ sudo apt-get install -y lxc-docker
```

versionを確認します。

``` bash
$ docker -v
Docker version 1.1.2, build d84a070
```

sudo なしでdockerコマンドを使えるようにします。

``` bash
$ sudo usermod -aG docker $USER
$ exit
```

### デプロイ

Ubuntu14.04のDockerにSSH接続します。

docker-registryの確認をします。

``` bash
$ curl 10.1.2.164:5000
"docker-registry server (dev) (v0.7.3)"
```

デプロイするイメージをpullします。

``` bash
$ docker pull 10.1.2.164:5000/rstudio-data
$ docker pull 10.1.2.164:5000/rstudio-server
$ docker pull 10.1.2.164:5000/openresty
```

さっそくコンテナを起動します。
rstudio-serverは、`--volumes-from`オプションで`/home/rstudio`ディレクトリをマウントしています。

``` bash
$ docker run -i -t -d --name rstudio-data 10.1.2.164:5000/rstudio-data
$ docker run -i -t -d --name rstudio-server --volumes-from rstudio-data 10.1.2.164:5000/rstudio-server /sbin/my_init
$ docker run -i -t -d --name openresty -p 80:80 -p 443:443 10.1.2.164:5000/openresty
```

### redis-cliでルーティングの設定

これまでIPアドレスの確認は、inspectとjqを使っていましたが、`-f`フラグを使うとGoのtemplateでフォーマットできます。

``` bash
$ docker inspect -f "&#123;&#123; .NetworkSettings.IPAddress &#125;&#125;"  12947ce0876e
172.17.0.3
```

`masato-rstudio.10.1.3.98.xip.io`のURLを、
`RStudio Server`コンテナの`172.17.0.3:8787`にルーティングします。

``` bash
$ docker run -it --rm --link openresty:redis dockerfile/redis bash -c 'redis-cli -h $REDIS_PORT_6379_TCP_ADDR set masato-rstudio.10.1.3.98.xip.io 172.17.0.3:8787'
```

### デプロイ用のスクリプト

`RStudio Server`の学習環境は複数必要なので、スクリプトを書いて自動化できるようにします。


``` bash ~/bin/deploy-rstudio.sh
#!/bin/bash
set -eo pipefail
if [ -z "$1" ]
then
  echo "Usage: deploy-rstudio username"
  exit -1
fi
REGISTRY="10.1.2.164:5000"
DOCKER_HOST="10.1.2.244"
RSTUDIO_PORT="8787"
docker run -i -t -d --name $1-rstudio-data ${REGISTRY}/rstudio-data
CONTAINER_ID=$(docker run -i -t -d --name $1-rstudio-server --volumes-from $1-rstudio-data ${REGISTRY}/rstudio-server /sbin/my_init)
CONTAINER_IP=$(docker inspect -f "&#123;&#123; .NetworkSettings.IPAddress &#125;&#125;" ${CONTAINER_ID})
REDIS_CLI='redis-cli -h $REDIS_PORT_6379_TCP_ADDR'
URL="$1-rstudio.${DOCKER_HOST}.xip.io"
REDIS_CMD="set ${URL} ${CONTAINER_IP}:${RSTUDIO_PORT}"
docker run -it --rm --link openresty:redis dockerfile/redis bash -c "${REDIS_CLI} ${REDIS_CMD}"
echo ${URL}
```

引数に作成したい名前を渡します。

``` bash
$ ~/bin/deploy-rstudio.sh test
c50de874da02d3f6254cd8d13935996897d75ff31a4c57cb3790e650956707a8
OK
test-rstudio.10.1.2.244.xip.io
```

ブラウザで確認します。

{% img center /2014/08/09/docker-analytic-sandbox-rstudio-openresty-deploy/test-rstudio.png %}


### まとめ

`RStudio Server`を使ってホームディレクトリに作成したRファイルは`Data-onlyコンテナ`に保存されます。
次は`Data-onlyコンテナ`は定期的にアーカイブして保存する仕組みを用意したいと思います。