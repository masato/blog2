title: "DockerのHTTP Routing - Part6: confd はじめに"
date: 2014-07-27 22:32:02
tags:
 - HTTPRouting
 - Nginx
 - nginx-proxy
 - fleetctl
 - confd
 - etcd
 - Consul
 - DockerRemoteAPI
description: Part1でリストしたconfdの使用例がCoreOSをfleetctlから操作する - Part2 hello.serviceで参考にしているExperimenting with CoreOS, confd, etcd, fleet, and CloudFormationの後半に出てきました。今回はconfdの使い方を見ていきます。いろいろ考えていくとconfdの仕組みが一番わかりやすそうです。
---

* `Update 2014-08-05`: [DockerのHTTP Routing - Part8: xip.io と Nginx と confd 0.6 で動的リバースプロキシ](/2014/08/05/docker-reverse-proxy-xipio-nginx-confd-sinatra/)

[Part1](/2014/07/14/docker-service-discovery-http-routing/)でリストしたconfdの使用例が[CoreOSをfleetctlから操作する - Part2: hello.service](/2014/07/26/coreos-fleetctl-hello-service/)で参考にしている[Experimenting with CoreOS, confd, etcd, fleet, and CloudFormation](http://marceldegraaf.net/2014/04/24/experimenting-with-coreos-confd-etcd-fleet-and-cloudformation.html)の後半に出てきました。今回はconfdの使い方を見ていきます。いろいろ考えていくとconfdの仕組みが一番わかりやすそうです。

<!-- more -->

### nginx.serviceのUnit

[marceldegraaf/nginx](https://registry.hub.docker.com/u/marceldegraaf/nginx/)をCoreOSのUnitに登録するserviceファイルです。

``` bash nginx.service
[Unit]
Description=nginx

[Service]
EnvironmentFile=/etc/environment
ExecStartPre=/usr/bin/docker pull marceldegraaf/nginx
ExecStart=/usr/bin/docker run --rm --name nginx -p 80:80 -e HOST_IP=${COREOS_PRIVATE_IPV4} marceldegraaf/nginx
ExecStop=/usr/bin/docker kill nginx

[X-Fleet]
X-Conflicts=nginx.service
```

fleetctlを使いnginx.serviceをUnitに登録します。

``` bash
$ fleetctl submit nginx.service
$ fleetctl start nginx.service
```

### confd

confdはetcdやConsulのキーを監視します。起動時に[TOML](https://github.com/mojombo/toml)形式の設定ファイルを`-config-file`で指定します。
キーの変更をトリガーにして、TOMLファイルに指定した処理を実行します。

* `src`に指定したテンプレートから、`dest`に指定したconfファイルを生成する。
* `dest`にconfファイルを生成後に、`reload_cmd`に指定したコマンドを実行する。

今回の例では、Nginxの設定ファイルが再作成され、Nginxがreloadされます。

``` ini /etc/confd/conf.d/nginx.toml
[template]
keys        = [ "app/server" ]
owner       = "nginx"
mode        = "0644"
src         = "nginx.conf.tmpl"
dest        = "/etc/nginx/sites-enabled/app.conf"
check_cmd   = "/usr/sbin/nginx -t -c /etc/nginx/nginx.conf"
reload_cmd  = "/usr/sbin/service nginx reload"
```

### marceldegraaf/nginx

[marceldegraaf/nginx](https://registry.hub.docker.com/u/marceldegraaf/nginx/)のDockerfileです。

``` bash Dockerfile
...
# Add boot script
ADD ./boot.sh /opt/boot.sh
RUN chmod +x /opt/boot.sh

# Run the boot script
CMD /opt/boot.sh
```

コンテナが起動すると、`/opt/boot.sh`を実行して以下の処理を行います。

* confdの監視登録
* confdをバックエンドで起動
* Nginxの起動

``` bash /opt/boot.sh
...
# Run confd in the background to watch the upstream servers
confd -interval 10 -node $ETCD -config-file /etc/confd/conf.d/nginx.toml &
echo "[nginx] confd is listening for changes on etcd..."
...
```

### まとめ

DockerクラスタをCoreOSで構築する場合にはconfdを使うと動的なリバースプロキシが便利に作れます。
一方でシングルノードのDockerホストの場合は、`Docker Remote API`の[GET /events](http://docs.docker.com/reference/api/docker_remote_api_v1.10/#monitor-dockers-events)からpollingかstreamingで通知を受ける方法もあります。

[Part1](/2014/07/14/docker-service-discovery-http-routing/)では、後者の方がミニマルで簡単のように思いましたが、どこかで通知を受け、その後のコマンドを実行するプロセスを管理する必要があります。

confdはこの目的に作られているので、TOML形式の設定ファイルで何をしているのか非常にわかりやすくまとまります。





