title: "CoreOSのfleetからfigに移行する - Part3: NginxとSSL証明書"
date: 2014-11-27 20:32:36
tags:
 - fig
 - Ubuntu
 - MoinMoin
 - Nginx
 - SSL
 - IDCFクラウド
description: これまではOpenRestyを使い動的にHTTP Routingをしてきましたが、figへの変更にあわせて単純なNginxのリバースプロキシと、起動時にsedでlinkされた環境変数を置換する方法にします。またNgixの設定ファイルやSSL証明書もイメージに入れていたのでプライベートのレジストリを使っていました。Docker Hub Registryに公開するため、SSL証明書などはDockerホストのディレクトリに保存するように変更します。fleetのunitファイルに比べて、リファクタリングしたこともありfig.ymlはかなりスッキリしました。
---

これまではOpenRestyを使い動的にHTTP Routingをしてきましたが、figへの変更にあわせて単純なNginxのリバースプロキシと、起動時にsedでlinkされた環境変数を置換する方法にします。またNgixの設定ファイルやSSL証明書もイメージに入れていたのでプライベートのレジストリを使っていました。Docker Hub Registryに公開するため、SSL証明書などはDockerホストのディレクトリに保存するように変更します。fleetのunitファイルに比べて、リファクタリングしたこともありfig.ymlはかなりスッキリしました。

<!-- more -->


### fig.yml

Dockerホストに`/opt/nginx/`ディレクトリを作成します。`/opt/nginx/certs`には証明書を配置します。`/opt/nginx/sites-enabled`ディレクトリには、[masato/nginx-rp](https://registry.hub.docker.com/u/masato/nginx-rp/)コンテナの起動時にテンプレートから生成するdefaultファイルが配置されます。ここでlinkされたMoinMoinコンテナのIPアドレスを置換してupstreamに指定します。

``` yml fig.yml
moinmoin:
  image: masato/moinmoin
  command: FooBar
  volumes_from:
    - data
  environment:
    VIRTUAL_HOST: example.com
nginx:
  image: masato/nginx-rp
  links:
    - moinmoin:web
  volumes:
    - /opt/nginx/certs:/etc/nginx/certs
    - /opt/nginx/sites-enabled:/etc/nginx/sites-enabled
    - /opt/nginx/log:/var/log/nginx
  ports:
    - "80:80"
    - "443:443"
data:
  image: ubuntu:14.04
  command: /bin/bash
  volumes:
    - /data
```

### superuserの作成

`command: FooBar`で指定しているユーザーはMoinMoinのsuperuserになります。このMoinMoinの唯一の接続ユーザーになります。docker execでコンテナに接続してユーザーファイルを作成します。

``` bash
$ docker exec -it root_moinmoin_1 bash
$ PYTHONPATH=/usr/local/share/moin /usr/local/bin/moin account create --name=FooBar --email=foo@bar.com --password=password
```

`/data/moin/user/`以下に、 ##########.##.#####のフォーマットでユーザーファイルが作成されるので、権限を変更して使えるようにします。

``` bash
$ chown www-data /data/moin/user/##########.##.#####
$ rm -fr /data/moin/user/cache
```

### Nginx Reverse Proxy

[masato/nginx-rp](https://registry.hub.docker.com/u/masato/nginx-rp/)にDockerイメージを公開しました。 Automated BuildのGitHubは[masato/nginx-rp](https://github.com/masato/nginx-rp)です。README.mdはちゃんと書かないといけないです。


